<!DOCTYPE html>
<html lang="zh-TW">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>班級聯絡簿</title>
    <script src="https://www.gstatic.com/firebasejs/8.10.1/firebase-app.js"></script>
    <script src="https://www.gstatic.com/firebasejs/8.10.1/firebase-database.js"></script>
    <script src="https://www.gstatic.com/firebasejs/8.10.1/firebase-auth.js"></script>
    
    <style>
        /* --- CSS 樣式區 (美編看這裡) --- */
        :root {
            --primary-green: #26D07C; /* LINE Green */
            --bg-gray: #eceef0;
            --bubble-gray: #ffffff;
            --bubble-green: #9de694;
        }
        body { margin: 0; padding: 0; font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif; background-color: var(--bg-gray); height: 100vh; overflow: hidden; }
        
        /* 通用元件 */
        .hidden { display: none !important; }
        .full-screen { position: absolute; top: 0; left: 0; width: 100%; height: 100%; display: flex; flex-direction: column; background: white; }
        
        /* 1. 登入頁 */
        #login-page { justify-content: center; align-items: center; background: #fff; }
        .role-btn { width: 150px; height: 150px; border-radius: 50%; border: none; font-size: 20px; font-weight: bold; margin: 20px; cursor: pointer; color: white; box-shadow: 0 4px 10px rgba(0,0,0,0.2); transition: transform 0.1s; }
        .btn-parent { background: #FF9F43; } /* 橘色 */
        .btn-teacher { background: #54a0ff; } /* 藍色 */
        .role-btn:active { transform: scale(0.95); }
        .input-group { display: flex; flex-direction: column; align-items: center; gap: 10px; margin-top: 20px; }
        input { padding: 10px; border: 1px solid #ddd; border-radius: 8px; font-size: 16px; width: 200px; text-align: center; }
        .confirm-btn { padding: 10px 30px; background: #333; color: white; border: none; border-radius: 20px; cursor: pointer; }

        /* 2. 聊天室 (核心) */
        #chat-header { height: 60px; background: #2d3436; color: white; display: flex; align-items: center; justify-content: space-between; padding: 0 15px; box-shadow: 0 2px 5px rgba(0,0,0,0.1); z-index: 10; }
        .header-title { font-size: 18px; font-weight: bold; }
        .call-btn { background: #26D07C; border: none; padding: 5px 15px; border-radius: 15px; color: white; font-size: 14px; cursor: pointer; display: flex; align-items: center; gap: 5px; text-decoration: none; }
        
        #messages-container { flex: 1; overflow-y: auto; padding: 15px; padding-bottom: 80px; display: flex; flex-direction: column; gap: 10px; }
        
        .message-row { display: flex; width: 100%; margin-bottom: 5px; }
        .msg-left { justify-content: flex-start; }
        .msg-right { justify-content: flex-end; }
        
        .bubble { max-width: 70%; padding: 10px 14px; border-radius: 15px; font-size: 15px; line-height: 1.4; position: relative; word-wrap: break-word; }
        .msg-left .bubble { background: var(--bubble-gray); border-top-left-radius: 2px; color: #333; }
        .msg-right .bubble { background: var(--bubble-green); border-top-right-radius: 2px; color: #000; }
        
        .msg-img { max-width: 100%; border-radius: 10px; margin-top: 5px; display: block; }
        .time-tag { font-size: 10px; color: #888; margin: 0 5px; align-self: flex-end; }

        /* 底部輸入框 */
        #input-area { position: fixed; bottom: 0; left: 0; width: 100%; background: white; padding: 10px; box-sizing: border-box; display: flex; align-items: center; gap: 10px; border-top: 1px solid #ddd; }
        #msg-input { flex: 1; padding: 10px; border-radius: 20px; border: 1px solid #ddd; outline: none; background: #f0f0f0; }
        .icon-btn { font-size: 24px; cursor: pointer; background: none; border: none; padding: 0 5px; }

        /* 3. 老師後台 */
        #admin-container { display: flex; height: 100%; }
        #user-list { width: 300px; background: white; border-right: 1px solid #ddd; overflow-y: auto; }
        .user-item { padding: 15px; border-bottom: 1px solid #f0f0f0; cursor: pointer; display: flex; justify-content: space-between; align-items: center; }
        .user-item:hover, .user-item.active { background: #f5f5f5; }
        .unread-dot { width: 10px; height: 10px; background: red; border-radius: 50%; display: none; }
        .unread .unread-dot { display: block; }
        
        /* 手機版後台適應 */
        @media (max-width: 768px) {
            #user-list { width: 100%; }
            #chat-view { display: none; position: fixed; top: 0; left: 0; width: 100%; height: 100%; z-index: 20; background: var(--bg-gray); }
            #chat-view.active { display: flex; flex-direction: column; }
            #back-btn { display: block !important; }
        }
        #back-btn { display: none; margin-right: 10px; font-size: 20px; cursor: pointer; }

    </style>
</head>
<body>

    <div id="login-page" class="full-screen">
        <h2 style="color:#555;">請選擇身份</h2>
        <div>
            <button class="role-btn btn-parent" onclick="showLoginInput('parent')">我是<br>家長</button>
            <button class="role-btn btn-teacher" onclick="showLoginInput('teacher')">我是<br>老師</button>
        </div>
        
        <div id="parent-input" class="input-group hidden">
            <p>請輸入您的名字 (例如: 大寶媽)</p>
            <input type="text" id="parent-name" placeholder="您的稱呼">
            <button class="confirm-btn" onclick="handleParentLogin()">進入聊天</button>
        </div>

        <div id="teacher-input" class="input-group hidden">
            <p>請輸入教師代碼</p>
            <input type="password" id="teacher-code" placeholder="代碼 (預設: 1234)">
            <button class="confirm-btn" onclick="handleTeacherLogin()">進入後台</button>
        </div>
    </div>

    <div id="parent-chat-ui" class="full-screen hidden">
        <div id="chat-header">
            <div style="display:flex; align-items:center;">
                <span class="header-title">班級老師</span>
            </div>
            <a href="#" id="call-link" class="call-btn" target="_blank">
                📞 通話
            </a>
        </div>
        <div id="parent-messages" class="messages-container">
            </div>
        <div id="input-area">
            <button class="icon-btn" onclick="document.getElementById('img-upload').click()">📷</button>
            <input type="file" id="img-upload" accept="image/*" style="display:none" onchange="handleImageUpload(this)">
            <input type="text" id="msg-text" placeholder="輸入訊息...">
            <button class="icon-btn" onclick="sendMessage()" style="color:var(--primary-green);">➤</button>
        </div>
    </div>

    <div id="admin-ui" class="full-screen hidden">
        <div id="admin-container">
            <div id="user-list">
                <div style="padding:15px; background:#f0f0f0; font-weight:bold;">家長列表</div>
                <div id="roster">
                    </div>
            </div>
            <div id="chat-view" class="full-screen" style="position:relative; background: var(--bg-gray);">
                <div id="chat-header">
                    <div style="display:flex; align-items:center;">
                        <span id="back-btn" onclick="closeAdminChat()">⬅</span>
                        <span id="admin-chat-title" class="header-title">請選擇家長</span>
                    </div>
                    <a href="#" id="admin-call-link" class="call-btn hidden" target="_blank">📞 加入通話</a>
                </div>
                <div id="admin-messages" class="messages-container" style="flex:1; overflow-y:auto; padding:15px; padding-bottom:80px;"></div>
                <div id="input-area" style="position:absolute;">
                    <button class="icon-btn" onclick="document.getElementById('admin-img-upload').click()">📷</button>
                    <input type="file" id="admin-img-upload" accept="image/*" style="display:none" onchange="handleImageUpload(this)">
                    <input type="text" id="admin-msg-text" placeholder="回覆訊息...">
                    <button class="icon-btn" onclick="sendAdminMessage()" style="color:#54a0ff;">➤</button>
                </div>
            </div>
        </div>
    </div>

    <script>
        // ==========================================
        //  設定區：請在這裡貼上你的 Firebase Config
        // ==========================================
        const firebaseConfig = {
            apiKey: "你的_API_KEY",
            authDomain: "你的專案.firebaseapp.com",
            databaseURL: "https://chat-1b478-default-rtdb.firebaseio.com",
            projectId: "你的專案ID",
            storageBucket: "你的專案.appspot.com",
            messagingSenderId: "...",
            appId: "..."
        };
        
        // 初始化 Firebase
        if (!firebase.apps.length) {
            firebase.initializeApp(firebaseConfig);
        }
        const db = firebase.database();
        const auth = firebase.auth();

        // 狀態變數
        let currentUser = null; // { uid, name, role }
        let currentChatUser = null; // 老師正在跟誰聊

        // --- 0. 初始化檢查 (是否已登入) ---
        window.onload = function() {
            const savedUser = localStorage.getItem('school_chat_user');
            if (savedUser) {
                currentUser = JSON.parse(savedUser);
                if (currentUser.role === 'parent') {
                    initParentUI();
                } else {
                    initAdminUI();
                }
            }
        };

        function showLoginInput(role) {
            document.getElementById('parent-input').classList.add('hidden');
            document.getElementById('teacher-input').classList.add('hidden');
            if(role === 'parent') document.getElementById('parent-input').classList.remove('hidden');
            if(role === 'teacher') document.getElementById('teacher-input').classList.remove('hidden');
        }

        // --- 1. 家長登入邏輯 ---
        function handleParentLogin() {
            const name = document.getElementById('parent-name').value;
            if (!name) return alert("請輸入名字");

            // 匿名登入 Firebase
            auth.signInAnonymously().then((cred) => {
                const uid = cred.user.uid;
                currentUser = { uid: uid, name: name, role: 'parent' };
                
                // 存入 LocalStorage (下次免登入)
                localStorage.setItem('school_chat_user', JSON.stringify(currentUser));
                
                // 更新資料庫的使用者資料
                db.ref('users/' + uid).update({
                    name: name,
                    role: 'parent',
                    last_seen: Date.now()
                });

                initParentUI();
            }).catch((error) => {
                console.error(error);
                alert("登入失敗: " + error.message);
            });
        }

        function initParentUI() {
            document.getElementById('login-page').classList.add('hidden');
            document.getElementById('parent-chat-ui').classList.remove('hidden');
            
            // 設定通話連結 (Jitsi)
            const callUrl = `https://meet.jit.si/ClassRoom-${currentUser.uid}`;
            document.getElementById('call-link').href = callUrl;

            // 監聽訊息
            loadMessages(currentUser.uid, 'parent-messages');
        }

        // --- 2. 老師登入邏輯 ---
        function handleTeacherLogin() {
            const code = document.getElementById('teacher-code').value;
            if (code !== '1234') return alert("代碼錯誤"); // 這裡可以改密碼

            // 老師也用匿名登入，但標記為 Admin
            auth.signInAnonymously().then((cred) => {
                currentUser = { uid: 'TEACHER_ADMIN', name: '老師', role: 'teacher' };
                localStorage.setItem('school_chat_user', JSON.stringify(currentUser));
                initAdminUI();
            });
        }

        function initAdminUI() {
            document.getElementById('login-page').classList.add('hidden');
            document.getElementById('admin-ui').classList.remove('hidden');
            loadUserList();
        }

        // --- 3. 訊息發送與接收邏輯 ---
        
        // 渲染單條訊息
        function renderMessage(msg, containerId, isMyMsg) {
            const container = document.getElementById(containerId);
            const div = document.createElement('div');
            div.className = `message-row ${isMyMsg ? 'msg-right' : 'msg-left'}`;
            
            let contentHtml = '';
            if (msg.image) {
                contentHtml = `<img src="${msg.image}" class="msg-img">`;
            } else {
                contentHtml = msg.text;
            }

            // 時間格式
            const time = new Date(msg.timestamp).toLocaleTimeString([], {hour: '2-digit', minute:'2-digit'});

            div.innerHTML = `
                ${isMyMsg ? `<div class="time-tag">${time}</div>` : ''}
                <div class="bubble">${contentHtml}</div>
                ${!isMyMsg ? `<div class="time-tag">${time}</div>` : ''}
            `;
            container.appendChild(div);
            container.scrollTop = container.scrollHeight; // 捲動到底部
        }

        // 監聽 Firebase 訊息
        function loadMessages(targetUid, containerId) {
            const container = document.getElementById(containerId);
            container.innerHTML = ''; // 清空

            db.ref('messages/' + targetUid).on('child_added', (snapshot) => {
                const msg = snapshot.val();
                // 判斷是誰發的
                let isMine = false;
                if (currentUser.role === 'parent' && msg.sender === 'parent') isMine = true;
                if (currentUser.role === 'teacher' && msg.sender === 'teacher') isMine = true;
                
                renderMessage(msg, containerId, isMine);
            });
        }

        // 家長發訊息
        function sendMessage() {
            const input = document.getElementById('msg-text');
            const text = input.value;
            if (!text) return;

            db.ref('messages/' + currentUser.uid).push({
                text: text,
                sender: 'parent',
                timestamp: Date.now()
            });
            
            // 更新最後訊息時間 (給老師列表排序用)
            db.ref('users/' + currentUser.uid).update({
                last_msg_time: Date.now(),
                last_msg_preview: text,
                has_unread: true
            });

            input.value = '';
        }

        // 老師發訊息
        function sendAdminMessage() {
            if (!currentChatUser) return;
            const input = document.getElementById('admin-msg-text');
            const text = input.value;
            if (!text) return;

            db.ref('messages/' + currentChatUser.uid).push({
                text: text,
                sender: 'teacher',
                timestamp: Date.now()
            });

            input.value = '';
        }

        // --- 4. 圖片上傳 (轉 Base64) ---
        function handleImageUpload(inputElement) {
            const file = inputElement.files[0];
            if (!file) return;
            
            // 限制大小 1MB (Realtime DB 不能存太大)
            if (file.size > 1024 * 1024) return alert("圖片太大，請選小一點的");

            const reader = new FileReader();
            reader.onload = function(e) {
                const base64 = e.target.result;
                const senderRole = currentUser.role;
                const targetUid = (senderRole === 'parent') ? currentUser.uid : currentChatUser.uid;

                db.ref('messages/' + targetUid).push({
                    image: base64,
                    sender: senderRole,
                    timestamp: Date.now()
                });
            };
            reader.readAsDataURL(file);
        }

        // --- 5. 老師列表邏輯 ---
        function loadUserList() {
            const roster = document.getElementById('roster');
            db.ref('users').on('value', (snapshot) => {
                roster.innerHTML = '';
                const users = [];
                snapshot.forEach(child => users.push({key: child.key, ...child.val()}));
                
                // 排序：有未讀的排前面，接著按時間排
                users.sort((a, b) => (b.last_msg_time || 0) - (a.last_msg_time || 0));

                users.forEach(u => {
                    if (u.role === 'teacher') return; // 不顯示老師自己
                    
                    const div = document.createElement('div');
                    div.className = 'user-item ' + (u.has_unread ? 'unread' : '');
                    div.innerHTML = `
                        <div>
                            <strong>${u.name}</strong><br>
                            <span style="font-size:12px; color:#888;">${u.last_msg_preview || '無訊息'}</span>
                        </div>
                        <div class="unread-dot"></div>
                    `;
                    div.onclick = () => openAdminChat(u);
                    roster.appendChild(div);
                });
            });
        }

        function openAdminChat(user) {
            currentChatUser = user;
            
            // 手機版 UI 切換
            if (window.innerWidth <= 768) {
                document.getElementById('chat-view').classList.add('active');
            }

            document.getElementById('admin-chat-title').innerText = user.name;
            document.getElementById('admin-call-link').classList.remove('hidden');
            document.getElementById('admin-call-link').href = `https://meet.jit.si/ClassRoom-${user.key}`;
            
            // 標記已讀
            db.ref('users/' + user.key).update({ has_unread: false });

            loadMessages(user.key, 'admin-messages');
        }

        function closeAdminChat() {
            document.getElementById('chat-view').classList.remove('active');
            currentChatUser = null;
        }

    </script>
</body>
</html># chattest
