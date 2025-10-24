<!DOCTYPE html>
<html lang="ja">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>鬼ごっこ 役割ランダム決定システム</title>
    
    <style>
        body {
            font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
            background-color: #f4f7f6;
            color: #333;
            padding: 20px;
            text-align: center;
        }

        h1 {
            color: #007bff;
        }

        h2 {
            border-bottom: 2px solid #ccc;
            padding-bottom: 10px;
            margin-top: 30px;
        }

        #settings-screen, #check-screen {
            max-width: 600px;
            margin: 20px auto;
            padding: 25px;
            background-color: #fff;
            border-radius: 10px;
            box-shadow: 0 4px 10px rgba(0, 0, 0, 0.1);
        }

        .role-input-group {
            display: flex;
            justify-content: space-between;
            margin-bottom: 15px;
            align-items: center;
        }

        .role-input-group input[type="text"] {
            flex-grow: 2;
            padding: 8px;
            margin-right: 10px;
            border: 1px solid #ccc;
            border-radius: 4px;
        }

        .role-input-group input[type="number"] {
            width: 60px;
            padding: 8px;
            border: 1px solid #ccc;
            border-radius: 4px;
            text-align: center;
        }

        button {
            background-color: #007bff;
            color: white;
            border: none;
            padding: 10px 20px;
            margin-top: 10px;
            border-radius: 5px;
            cursor: pointer;
            font-size: 16px;
            transition: background-color 0.3s;
        }

        button:hover:not(:disabled) {
            background-color: #0056b3;
        }

        button:disabled {
            background-color: #ccc;
            cursor: not-allowed;
        }

        #role-display {
            margin-top: 20px;
            padding: 20px;
            border: 2px dashed #ffc107;
            background-color: #fff3cd;
            border-radius: 8px;
        }

        .role-name {
            font-size: 32px;
            font-weight: bold;
            color: #dc3545; /* 重要な役割を目立たせる色 */
            margin: 10px 0;
        }

        .error {
            color: red;
            font-weight: bold;
            margin-top: 10px;
        }
    </style>
</head>
<body>
    <h1>鬼ごっこ 役割ランダム決定システム</h1>

    <div id="settings-screen">
        <h2>役割と人数の設定</h2>
        <div id="roles-container">
            </div>

        <button onclick="addRoleInput()">役割を追加</button>
        <p>合計参加人数: <span id="total-participants">0</span> 人</p>
        <button id="start-button" onclick="startGame()" disabled>役割を決定し、開始</button>
        <p id="error-message" class="error"></p>
    </div>

    <div id="check-screen" style="display:none;">
        <h2 id="participant-number">参加者 1 の役割確認</h2>
        <p>他の人に見られないように、画面を下に渡してください。</p>
        <button id="check-role-button" onclick="showRole()">役割を確認する</button>

        <div id="role-display" style="display:none;">
            <h3>あなたの役割は...</h3>
            <p id="assigned-role" class="role-name"></p>
        </div>

        <button id="next-button" onclick="nextParticipant()" style="display:none;">次の参加者へ</button>
        <button id="reset-button" onclick="resetGame()" style="display:none; margin-top: 10px;">最初からやり直す</button>
    </div>

    <script>
        let roles = []; // 確定した役割リスト (例: ['鬼', '鬼', '逃走者', ...])
        let participantCount = 0; // 現在の参加者番号
        const rolesContainer = document.getElementById('roles-container');
        const totalParticipantsSpan = document.getElementById('total-participants');
        const startButton = document.getElementById('start-button');
        const errorMessage = document.getElementById('error-message');

        // --- 役割設定画面のロジック ---

        /**
         * 役割入力フィールドを動的に追加する
         */
        function addRoleInput() {
            const roleGroup = document.createElement('div');
            roleGroup.classList.add('role-input-group');
            roleGroup.innerHTML = `
                <input type="text" placeholder="役割名 (例: 鬼)" class="role-name-input" oninput="updateTotalParticipants()">
                <input type="number" placeholder="人数" min="1" value="1" class="role-count-input" oninput="updateTotalParticipants()">
            `;
            rolesContainer.appendChild(roleGroup);
            updateTotalParticipants(); // 初期値で更新
        }

        /**
         * 現在の役割の合計人数を計算し、表示を更新する
         */
        function updateTotalParticipants() {
            const roleNames = document.querySelectorAll('.role-name-input');
            const roleCounts = document.querySelectorAll('.role-count-input');
            let total = 0;
            let allValid = true;

            for (let i = 0; i < roleCounts.length; i++) {
                const count = parseInt(roleCounts[i].value);
                const name = roleNames[i].value.trim();

                if (isNaN(count) || count < 1 || name === "") {
                    allValid = false;
                }
                if (!isNaN(count)) {
                    total += count;
                }
            }

            totalParticipantsSpan.textContent = total;

            // 役割が1つ以上定義され、合計人数が1人以上で、全ての入力が有効ならボタンを有効化
            if (total >= 1 && roleNames.length >= 1 && allValid) {
                startButton.disabled = false;
                errorMessage.textContent = '';
            } else if (roleNames.length === 0) {
                startButton.disabled = true;
                errorMessage.textContent = '役割を最低1つ設定してください。';
            } else {
                startButton.disabled = true;
                errorMessage.textContent = '役割名を入力し、人数を1人以上に設定してください。';
            }
        }

        /**
         * 役割設定を確定し、ゲームを開始する (役割の決定とシャッフル)
         */
        function startGame() {
            const roleNames = document.querySelectorAll('.role-name-input');
            const roleCounts = document.querySelectorAll('.role-count-input');
            let tempRoles = [];

            // 役割リストを作成
            for (let i = 0; i < roleNames.length; i++) {
                const name = roleNames[i].value.trim();
                const count = parseInt(roleCounts[i].value);

                if (name && count >= 1) {
                    for (let j = 0; j < count; j++) {
                        tempRoles.push(name);
                    }
                }
            }

            if (tempRoles.length === 0) {
                errorMessage.textContent = '役割と人数を正しく設定してください。';
                return;
            }

            // 役割をランダムにシャッフル
            roles = shuffleArray(tempRoles);
            participantCount = 1;

            // 画面切り替え
            document.getElementById('settings-screen').style.display = 'none';
            document.getElementById('check-screen').style.display = 'block';
            updateCheckScreen();
        }

        /**
         * 配列をシャッフルする (Fisher-Yatesアルゴリズム)
         * @param {Array} array - シャッフルする配列
         * @returns {Array} シャッフルされた新しい配列
         */
        function shuffleArray(array) {
            const newArray = [...array];
            for (let i = newArray.length - 1; i > 0; i--) {
                const j = Math.floor(Math.random() * (i + 1));
                [newArray[i], newArray[j]] = [newArray[j], newArray[i]];
            }
            return newArray;
        }

        // --- 役割確認画面のロジック ---

        /**
         * 役割確認画面の表示を更新する
         */
        function updateCheckScreen() {
            document.getElementById('participant-number').textContent = `参加者 ${participantCount} の役割確認`;
            document.getElementById('role-display').style.display = 'none';
            document.getElementById('check-role-button').style.display = 'block';
            document.getElementById('next-button').style.display = 'none';
            document.getElementById('reset-button').style.display = 'none';
        }

        /**
         * 自分の役割を表示する
         */
        function showRole() {
            const roleIndex = participantCount - 1;
            const assignedRole = roles[roleIndex];

            document.getElementById('assigned-role').textContent = assignedRole;
            document.getElementById('role-display').style.display = 'block';
            document.getElementById('check-role-button').style.display = 'none';

            // 最後の参加者でなければ「次へ」ボタンを表示
            if (participantCount < roles.length) {
                document.getElementById('next-button').style.display = 'block';
            } else {
                // 全員確認完了
                document.getElementById('next-button').style.display = 'none';
                document.getElementById('reset-button').style.display = 'block';
                document.getElementById('participant-number').textContent = `全員の確認が完了しました！`;
            }
        }

        /**
         * 次の参加者の確認画面へ進む
         */
        function nextParticipant() {
            participantCount++;
            updateCheckScreen();
        }

        /**
         * ゲームをリセットし、設定画面に戻る
         */
        function resetGame() {
            roles = [];
            participantCount = 0;
            // 画面切り替え
            document.getElementById('check-screen').style.display = 'none';
            document.getElementById('settings-screen').style.display = 'block';
            updateTotalParticipants(); // ボタンの状態を更新
        }

        // 初期化処理: ページ読み込み後に実行される
        document.addEventListener('DOMContentLoaded', () => {
            addRoleInput(); // 最初に役割入力フィールドを一つ追加
        });
    </script>
</body>
</html>
