# Jzip
### Jzip는 Jade의 가독성을 포기하고 
## 압축성만 챙겼습니다
# Jzip > Jade
# Jade > Jzip 
## 이렇게 변환이 가능합니다
### 원리는 코드를 통해 알아보세요!
코드
```<!DOCTYPE html>
<html lang="ko">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Jzip Ultra - 70%+ Text Compressor</title>
    <style>
        :root {
            --bg-color: #0d1117;
            --card-bg: #161b22;
            --accent: #58a6ff;
            --accent-hover: #1f6feb;
            --text-main: #c9d1d9;
            --text-sub: #8b949e;
            --border: #30363d;
            --success: #3fb950;
            --warning: #d29922;
        }

        * {
            box-sizing: border-box;
            margin: 0;
            padding: 0;
            font-family: 'Fira Code', 'Cascadia Code', Consolas, Monaco, monospace;
        }

        body {
            background-color: var(--bg-color);
            color: var(--text-main);
            min-height: 100vh;
            display: flex;
            flex-direction: column;
            padding: 1.5rem;
        }

        header {
            text-align: center;
            margin-bottom: 1.5rem;
        }

        header h1 {
            font-size: 2.2rem;
            color: var(--accent);
            margin-bottom: 0.3rem;
        }

        header p {
            color: var(--text-sub);
            font-size: 0.9rem;
        }

        .controls {
            display: flex;
            justify-content: center;
            gap: 1rem;
            margin-bottom: 1.5rem;
        }

        button {
            background-color: var(--accent);
            color: #fff;
            border: none;
            padding: 0.75rem 1.8rem;
            font-size: 0.95rem;
            font-weight: bold;
            border-radius: 6px;
            cursor: pointer;
            transition: all 0.2s ease;
        }

        button:hover {
            background-color: var(--accent-hover);
            transform: translateY(-1px);
        }

        button.secondary {
            background-color: transparent;
            border: 1px solid var(--accent);
            color: var(--accent);
        }

        button.secondary:hover {
            background-color: rgba(88, 166, 255, 0.1);
        }

        .container {
            display: grid;
            grid-template-columns: 1fr 1fr;
            gap: 1.2rem;
            flex: 1;
            max-width: 1400px;
            margin: 0 auto;
            width: 100%;
        }

        .panel {
            background-color: var(--card-bg);
            border: 1px solid var(--border);
            border-radius: 8px;
            display: flex;
            flex-direction: column;
        }

        .panel-header {
            padding: 0.8rem 1rem;
            background-color: rgba(0, 0, 0, 0.3);
            border-bottom: 1px solid var(--border);
            display: flex;
            justify-content: space-between;
            align-items: center;
        }

        .panel-title {
            font-weight: bold;
            font-size: 0.9rem;
            color: var(--accent);
        }

        .stats {
            font-size: 0.85rem;
            color: var(--text-sub);
        }

        .stats span {
            color: var(--success);
            font-weight: bold;
        }

        textarea {
            flex: 1;
            background: transparent;
            border: none;
            color: var(--text-main);
            padding: 1rem;
            font-size: 0.9rem;
            line-height: 1.5;
            resize: none;
            outline: none;
            min-height: 450px;
        }

        textarea::placeholder {
            color: var(--border);
        }

        @media (max-width: 768px) {
            .container {
                grid-template-columns: 1fr;
            }
        }
    </style>
</head>
<body>

    <header>
        <h1>⚡ Jzip Ultra (High-Ratio Text Compressor)</h1>
        <p>Jade 포맷 전용 고압축 텍스트 인코더 (목표 압축률: 60% ~ 75%+)</p>
    </header>

    <div class="controls">
        <button id="btnCompress">압축하기 (Compress ➔)</button>
        <button id="btnDecompress" class="secondary">복원하기 (Decompress ⬅)</button>
    </div>

    <div class="container">
        <!-- Input Panel -->
        <div class="panel">
            <div class="panel-header">
                <span class="panel-title">📄 Original Jade Text</span>
                <div class="stats" id="inputStats">0 Bytes</div>
            </div>
            <textarea id="inputText" placeholder="여기에 원본 Jade 문법 텍스트를 입력하세요..."></textarea>
        </div>

        <!-- Output Panel -->
        <div class="panel">
            <div class="panel-header">
                <span class="panel-title">📦 Jzip Compressed Text (수정/입력 가능)</span>
                <div class="stats" id="outputStats">0 Bytes</div>
            </div>
            <textarea id="outputText" placeholder="압축 결과가 여기에 출력됩니다. 직접 압축 코드를 붙여넣고 복원할 수도 있습니다..."></textarea>
        </div>
    </div>

    <script>
        class JzipUltraEngine {
            constructor() {
                // Base62 사전용 단축 키 문자 세트
                this.DICT_CHARS = "0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ";
            }

            toBase62(n) {
                if (n === 0) return '0';
                let res = '';
                while (n > 0) {
                    res = this.DICT_CHARS[n % 62] + res;
                    n = Math.floor(n / 62);
                }
                return res;
            }

            compress(text) {
                if (!text.trim()) return '';

                const lines = text.split('\n');
                const keyFreq = {};
                const cleanLines = [];

                // 1. 주석 및 불필요한 공백 제거 & 키 빈도 분석
                for (let rawLine of lines) {
                    let l = rawLine.split('#')[0].trimEnd();
                    if (!l.trim()) continue; // 주석/빈 줄 제외

                    // 키 추출
                    let m = l.match(/^\s*([a-zA-Z0-9_-]+)\s*:/) || l.match(/^\s*<([a-zA-Z0-9_-]+)>/);
                    if (m) {
                        let k = m[1];
                        keyFreq[k] = (keyFreq[k] || 0) + 1;
                    }
                    cleanLines.push(l);
                }

                // 2. 고빈도 키 사전(Dictionary) 생성 및 단축 기호($id) 부여
                const keys = Object.keys(keyFreq).sort((a, b) => keyFreq[b] - keyFreq[a]);
                const dict = {};
                let dictHeader = '';

                keys.forEach((k, idx) => {
                    // 키 길이가 짧아지는 경우에만 사전에 등록
                    const id = this.toBase62(idx);
                    if (k.length > id.length + 1) {
                        dict[k] = `@${id}`;
                        dictHeader += `${k}=${id};`;
                    }
                });

                // 3. 인라인 매핑 및 초고밀도 기호화
                let tokens = [];
                for (let line of cleanLines) {
                    let content = line.trim();

                    // 블록 닫기 마침표축약
                    if (content === '.') {
                        tokens.push('}');
                        continue;
                    }

                    // 불리언 / Null 특수 단축기호화
                    // <key>/ -> +key , <key> -> -key , <key>; -> ~key
                    if (content.startsWith('<')) {
                        content = content.replace(/<([a-zA-Z0-9_-]+)>\//, (_, k) => `+${dict[k] || k}`)
                                         .replace(/<([a-zA-Z0-9_-]+)>;/, (_, k) => `~${dict[k] || k}`)
                                         .replace(/<([a-zA-Z0-9_-]+)>/, (_, k) => `-${dict[k] || k}`);
                    } else {
                        // 키:값 매핑 (공백 완전제거)
                        content = content.replace(/^([a-zA-Z0-9_-]+)\s*:\s*/, (_, k) => `${dict[k] || k}:`);
                    }

                    // 배열 리스트 (-) 처리
                    if (content.startsWith('- ')) {
                        content = '!' + content.substring(2);
                    }

                    tokens.push(content);
                }

                let stream = tokens.join('|');

                // 4. 연속 닫기 괄호 '}' 수식 RLE 압축 (예: }|}|} -> }3)
                stream = stream.replace(/(\}\|){2,}/g, (m) => {
                    let count = m.split('|').length - 1;
                    return `}${count}|`;
                });

                // 5. 연속 구분자 및 리스트 수식 압축
                stream = stream.replace(/\|!/g, '!').replace(/\|/g, ';');

                return `JZ[${dictHeader}]${stream}`;
            }

            decompress(jzip) {
                if (!jzip.startsWith('JZ[')) return '⚠️ 올바른 Jzip Ultra 포맷이 아닙니다.';

                const headerEnd = jzip.indexOf(']');
                if (headerEnd === -1) return '⚠️ 헤더 파싱 오류!';

                const dictStr = jzip.substring(3, headerEnd);
                let body = jzip.substring(headerEnd + 1);

                // 1. 사전 복원
                const dict = {};
                if (dictStr) {
                    dictStr.split(';').forEach(p => {
                        if (!p) return;
                        const [k, id] = p.split('=');
                        dict[`@${id}`] = k;
                    });
                }

                const replaceKey = (str) => {
                    return str.replace(/@([0-9a-zA-Z]+)/g, (_, id) => dict[`@${id}`] || `@${id}`);
                };

                // 2. RLE 연속 닫기(}) 복원
                body = body.replace(/\}(\d+)/g, (_, c) => '};'.repeat(parseInt(c) - 1) + '}');

                // 3. 리스트 기호(!) 분리
                body = body.replace(/!/g, ';!');

                const tokens = body.split(';').filter(t => t.length > 0);
                let result = '';
                let indent = 0;

                for (let token of tokens) {
                    if (token === '}') {
                        indent = Math.max(0, indent - 4);
                        result += ' '.repeat(indent) + '.\n';
                        continue;
                    }

                    let line = token;

                    // 배열 항목
                    if (line.startsWith('!')) {
                        line = '- ' + line.substring(1);
                    } 
                    // 불리언/Null 복원
                    else if (line.startsWith('+')) {
                        line = `<${replaceKey(line.substring(1))}>/`;
                    } else if (line.startsWith('~')) {
                        line = `<${replaceKey(line.substring(1))}>;`;
                    } else if (line.startsWith('-') && !line.includes(':')) {
                        line = `<${replaceKey(line.substring(1))}>`;
                    } else {
                        // 키:값 구조 복원
                        line = replaceKey(line).replace(':', ' : ');
                    }

                    result += ' '.repeat(indent) + line + '\n';

                    // 하위 계층 자동 들여쓰기 처리
                    if (line.endsWith(' :')) {
                        indent += 4;
                    }
                }

                return result.trimEnd();
            }
        }

        // --- DOM 및 상호작용 ---
        const engine = new JzipUltraEngine();

        const inputText = document.getElementById('inputText');
        const outputText = document.getElementById('outputText');
        const inputStats = document.getElementById('inputStats');
        const outputStats = document.getElementById('outputStats');

        // 압축률 테스트용 대용량 샘플 Jade 데이터
        inputText.value = `# 게임 캐릭터 종합 데이터베이스
character :
    name : ArchMage
    class : Wizard
    level : 99
    
    stats :
        strength : 12
        agility : 45
        intelligence : 180
        mana : 1200
        health : 450
    .
    
    inventory :
        - Staff of Wisdom
        - Mana Potion
        - Teleport Scroll
        - Ring of Fire
    .
    
    skills :
        fireball :
            damage : 350
            cost : 40
        .
        icebolt :
            damage : 220
            cost : 25
        .
    .
    
    <isAlive>/
    <isVIP>/
    <hasPet>;
    <isBanned>
.`;

        function calculateStats() {
            const inBytes = new Blob([inputText.value]).size;
            const outBytes = new Blob([outputText.value]).size;

            inputStats.textContent = `${inBytes} Bytes`;

            if (outBytes > 0 && inBytes > 0) {
                const ratio = ((1 - (outBytes / inBytes)) * 100).toFixed(1);
                outputStats.innerHTML = `${outBytes} Bytes (<span>${ratio}% 압축 절감</span>)`;
            } else {
                outputStats.textContent = `0 Bytes`;
            }
        }

        document.getElementById('btnCompress').addEventListener('click', () => {
            outputText.value = engine.compress(inputText.value);
            calculateStats();
        });

        document.getElementById('btnDecompress').addEventListener('click', () => {
            // 오른쪽(outputText)의 압축 텍스트를 복원하여 왼쪽(inputText)에 대입
            if (!outputText.value.trim()) return;
            inputText.value = engine.decompress(outputText.value);
            calculateStats();
        });

        inputText.addEventListener('input', calculateStats);
        outputText.addEventListener('input', calculateStats);

        // 초기 실시간 압축 실행
        outputText.value = engine.compress(inputText.value);
        calculateStats();
    </script>
</body>
</html>```
