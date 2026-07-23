```import re
import re
import json
from typing import Any, Dict, List, Tuple, Union

class JadeParseError(Exception):
    """Jade 규범 위반 및 AI 파싱 실패 예외"""
    pass

class Jade:
    _KEY_REGEX = re.compile(r'^[a-zA-Z_][a-zA-Z0-9_-]*$')
    _BOOL_NULL_REGEX = re.compile(r'^<([a-zA-Z_][a-zA-Z0-9_-]*)>([/;]?)$')

    # ==========================================
    # 📥 파싱: Jade ➔ Python Dict
    # ==========================================
    @classmethod
    def loads(cls, jade_str: str) -> Dict[str, Any]:
        lines = jade_str.splitlines()
        cleaned_lines: List[Tuple[int, int, str]] = []

        for line_num, raw_line in enumerate(lines, 1):
            if '\t' in raw_line:
                raise JadeParseError(f"[Line {line_num}] 탭(Tab) 문자는 금지됩니다. 공백 4칸을 사용하세요.")
            
            line = cls._strip_comment(raw_line)
            stripped = line.strip()
            
            if not stripped:
                continue

            indent = len(line) - len(line.lstrip(' '))
            if indent % 4 != 0:
                raise JadeParseError(f"[Line {line_num}] 들여쓰기는 정확히 공백 4칸 단위여야 합니다. (현재: {indent}칸)")

            cleaned_lines.append((line_num, indent // 4, stripped))

        if not cleaned_lines:
            return {}

        root: Dict[str, Any] = {}
        stack: List[Tuple[Union[Dict, List], int, str]] = [(root, -1, 'object')]

        i = 0
        total_lines = len(cleaned_lines)

        while i < total_lines:
            line_num, level, text = cleaned_lines[i]

            # A. 블록 마감(.) 처리 (객체 블록만 마감)
            if text == '.':
                if len(stack) <= 1:
                    raise JadeParseError(f"[Line {line_num}] 상위 블록이 존재하지 않는 잘못된 마침표('.')입니다.")
                
                while len(stack) > 1 and stack[-1][1] >= level:
                    stack.pop()
                i += 1
                continue

            # B. 스택 수준 동기화
            while len(stack) > 1 and stack[-1][1] >= level:
                stack.pop()

            curr_container, _, curr_type = stack[-1]

            # C. Boolean / Null 처리 (<key>/, <key>, <key>;)
            if text.startswith('<') and '>' in text:
                match = cls._BOOL_NULL_REGEX.match(text)
                if match:
                    key, modifier = match.groups()
                    val = True if modifier == '/' else (None if modifier == ';' else False)

                    if curr_type == 'object':
                        curr_container[key] = val
                    elif curr_type == 'array':
                        curr_container.append(val)
                    i += 1
                    continue

            # D. 배열 요소 (- value)
            if text.startswith('- '):
                if curr_type != 'array':
                    raise JadeParseError(f"[Line {line_num}] 객체 블록 내에서 직접 '-' 배열 구문을 작성할 수 없습니다.")
                
                val_str = text[2:].strip()
                curr_container.append(cls._parse_value(val_str))
                i += 1
                continue

            # E. Key : Value 혹은 중첩 블록
            if ':' in text:
                key, val_str = map(str.strip, text.split(':', 1))
                
                if not cls._KEY_REGEX.match(key):
                    raise JadeParseError(f"[Line {line_num}] 유효하지 않은 키 이름 '{key}' 입니다.")

                if val_str == '':
                    if i + 1 < total_lines:
                        _, next_level, next_text = cleaned_lines[i + 1]
                        if next_level == level + 1:
                            is_next_array = next_text.startswith('- ')
                            new_container = [] if is_next_array else {}
                            new_type = 'array' if is_next_array else 'object'
                            
                            if curr_type == 'object':
                                curr_container[key] = new_container
                            elif curr_type == 'array':
                                curr_container.append({key: new_container})
                                
                            stack.append((new_container, level + 1, new_type))
                        else:
                            raise JadeParseError(f"[Line {line_num}] 블록 선언 키 '{key}' 다음 줄의 들여쓰기가 올바르지 않습니다.")
                    else:
                        raise JadeParseError(f"[Line {line_num}] 파일 끝에 닫히지 않은 블록 키 '{key}'가 있습니다.")
                else:
                    if curr_type == 'object':
                        curr_container[key] = cls._parse_value(val_str)
                    else:
                        raise JadeParseError(f"[Line {line_num}] 배열 내부에 direct key:value 선언은 불가능합니다.")
                i += 1
                continue

            raise JadeParseError(f"[Line {line_num}] 문법 오류: '{text}'")

        if len(stack) > 1:
            raise JadeParseError("파일 끝(EOF)에 도달했으나 마감되지 않은 객체 블록('.')이 존재합니다.")

        return root

    # ==========================================
    # 📤 직렬화: Python Dict ➔ Jade (배열 마감 마침표 버그 수정)
    # ==========================================
    @classmethod
    def dumps(cls, data: Union[Dict, List], indent_level: int = 0) -> str:
        lines = []
        indent = " " * (indent_level * 4)

        if isinstance(data, dict):
            for key, val in data.items():
                if isinstance(val, bool):
                    lines.append(f"{indent}<{key}>/" if val else f"{indent}<{key}>")
                elif val is None:
                    lines.append(f"{indent}<{key}>;")
                elif isinstance(val, dict):
                    # 객체(Dict) 블록만 마침표(.)로 닫음
                    lines.append(f"{indent}{key} :")
                    lines.append(cls.dumps(val, indent_level + 1))
                    lines.append(f"{indent}.")
                elif isinstance(val, list):
                    # 리스트(Array) 블록은 마침표(.)를 붙이지 않음!
                    lines.append(f"{indent}{key} :")
                    lines.append(cls.dumps(val, indent_level + 1))
                else:
                    val_str = cls._format_value(val)
                    lines.append(f"{indent}{key} : {val_str}")
                    
        elif isinstance(data, list):
            for item in data:
                if isinstance(item, dict):
                    sub_jade = cls.dumps(item, indent_level + 1)
                    sub_lines = sub_jade.splitlines()
                    if sub_lines:
                        lines.append(f"{indent}- {sub_lines[0].strip()}")
                        lines.extend(sub_lines[1:])
                elif isinstance(item, list):
                    lines.append(cls.dumps(item, indent_level + 1))
                else:
                    val_str = cls._format_value(item)
                    lines.append(f"{indent}- {val_str}")

        return "\n".join(lines)

    # ==========================================
    # 🛠️ 최적화 헬퍼 메서드
    # ==========================================
    @staticmethod
    def _strip_comment(line: str) -> str:
        if '#' not in line:
            return line
            
        in_quotes = False
        quote_char = ''
        i = 0
        n = len(line)
        
        while i < n:
            char = line[i]
            if char in ('"', "'"):
                if not in_quotes:
                    in_quotes = True
                    quote_char = char
                elif quote_char == char:
                    escapes = 0
                    j = i - 1
                    while j >= 0 and line[j] == '\\':
                        escapes += 1
                        j -= 1
                    if escapes % 2 == 0:
                        in_quotes = False
            elif char == '#' and not in_quotes:
                return line[:i]
            i += 1
            
        return line

    @staticmethod
    def _parse_value(val_str: str) -> Any:
        if (val_str.startswith('"') and val_str.endswith('"')) or \
           (val_str.startswith("'") and val_str.endswith("'")):
            raw = val_str[1:-1]
            return raw.replace('\\"', '"').replace("\\'", "'").replace('\\\\', '\\').replace('\\n', '\n')
        
        if val_str.isdigit() or (val_str.startswith('-') and len(val_str) > 1 and val_str[1:].isdigit()):
            return int(val_str)
            
        if re.match(r'^-?[0-9]+\.[0-9]+$', val_str):
            return float(val_str)
            
        return val_str

    @staticmethod
    def _format_value(val: Any) -> str:
        if isinstance(val, str):
            if any(c in val for c in (':', '.', '-', '#', '<', '>', '"', "'", ' ', '\n')):
                escaped = val.replace('\\', '\\\\').replace('"', '\\"').replace('\n', '\\n')
                return f'"{escaped}"'
            return val
        return str(val)

    # ==========================================
    # 🔌 API 인터페이스
    # ==========================================
    @classmethod
    def jade_to_json(cls, jade_str: str, indent: int = 2) -> str:
        return json.dumps(cls.loads(jade_str), indent=indent, ensure_ascii=False)

    @classmethod
    def json_to_jade(cls, json_str: str) -> str:
        return cls.dumps(json.loads(json_str))

    @classmethod
    def lint(cls, jade_str: str) -> Dict[str, Any]:
        try:
            cls.loads(jade_str)
            return {"valid": True, "error": None}
        except JadeParseError as e:
            return {"valid": False, "error": str(e)}```
