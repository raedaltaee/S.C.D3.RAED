<!DOCTYPE html>
<html lang="ar">
<head>
    <meta charset="UTF-8">
    <title>Firebase Authentication</title>
    <style>
        .container { max-width: 800px; margin: 0 auto; padding: 20px; }
        .message-box { 
            display: none; 
            padding: 15px; 
            margin-bottom: 20px; 
            border-radius: 4px; 
            border: 1px solid transparent;
        }
        .text-3xl { font-size: 1.875rem; }
        .font-bold { font-weight: 700; }
        .text-gray-800 { color: #1f2937; }
        .mb-6 { margin-bottom: 1.5rem; }
        .text-lg { font-size: 1.125rem; }
        .text-gray-700 { color: #374151; }
        .mb-4 { margin-bottom: 1rem; }
        .text-sm { font-size: 0.875rem; }
        .text-gray-500 { color: #6b7280; }
        .font-mono { font-family: monospace; }
    </style>
</head>
<body>
    <div class="container">
        <h1 class="text-3xl font-bold text-gray-800 mb-6">Firebase Authentication</h1>
        <div id="messageBox" class="message-box"></div>
        <p id="authStatus" class="text-lg text-gray-700 mb-4">Initializing Firebase...</p>
        <p class="text-sm text-gray-500">Your User ID: <span id="userIdDisplay" class="font-mono">Loading...</span></p>
    </div>

    <script type="module">
        // Import Firebase modules from CDN
        import { initializeApp } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-app.js";
        import { 
            getAuth, 
            signInAnonymously, 
            signInWithCustomToken, 
            onAuthStateChanged 
        } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-auth.js";
        import { 
            getFirestore, 
            collection, 
            doc, 
            setDoc, 
            onSnapshot 
        } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-firestore.js";

        // Global Firebase variables
        let app;
        let auth;
        let db;
        let userId;

        const messageBox = document.getElementById('messageBox');
        const authStatusElement = document.getElementById('authStatus');
        const userIdDisplayElement = document.getElementById('userIdDisplay');

        function showMessage(message, isError = false) {
            messageBox.textContent = message;
            messageBox.style.display = 'block';
            if (isError) {
                messageBox.style.backgroundColor = '#ffcdd2';
                messageBox.style.color = '#b71c1c';
                messageBox.style.borderColor = '#ef5350';
            } else {
                messageBox.style.backgroundColor = '#e8f5e9';
                messageBox.style.color = '#2e7d32';
                messageBox.style.borderColor = '#66bb6a';
            }
        }

        // Initialize Firebase and handle authentication
        window.onload = async function() {
            try {
                // 1. تأكد من وجود المتغيرات العالمية
                const appId = window.__app_id || 'default-app-id';
                
                // 2. حل مشكلة تحليل التكوين
                let firebaseConfig;
                try {
                    firebaseConfig = typeof __firebase_config === 'string' 
                        ? JSON.parse(__firebase_config) 
                        : __firebase_config;
                } catch (e) {
                    console.error("Error parsing Firebase config:", e);
                    showMessage("Invalid Firebase configuration format", true);
                    return;
                }
                
                const initialAuthToken = window.__initial_auth_token || null;

                // 3. التحقق من التكوين بشكل صارم
                if (!firebaseConfig || !firebaseConfig.apiKey) {
                    const errorMsg = "Firebase config is missing or invalid";
                    console.error(errorMsg, firebaseConfig);
                    showMessage(`${errorMsg}. Check your configuration.`, true);
                    return;
                }

                // 4. تهيئة Firebase
                app = initializeApp(firebaseConfig);
                auth = getAuth(app);
                db = getFirestore(app);

                // 5. إضافة معالج لخطأ المصادقة العامة
                auth.onAuthStateChanged(async (user) => {
                    try {
                        if (user) {
                            userId = user.uid;
                            authStatusElement.textContent = `Authenticated as: ${userId}`;
                            userIdDisplayElement.textContent = userId;
                            showMessage("Authentication successful!");

                            // 6. إصلاح مسار Firestore
                            const userDocRef = doc(db, `artifacts/${appId}/users/${userId}`, "profile");

                            // 7. إضافة بيانات اختبار
                            await setDoc(userDocRef, {
                                lastLogin: new Date().toISOString(),
                                appId: appId,
                                created: new Date().toISOString()
                            }, { merge: true });

                            // 8. إعداد مستمع في الوقت الحقيقي
                            onSnapshot(userDocRef, (docSnap) => {
                                if (docSnap.exists()) {
                                    console.log("Profile data:", docSnap.data());
                                } else {
                                    console.log("No profile data found");
                                }
                            }, (error) => {
                                console.error("Firestore listen error:", error);
                            });
                        } else {
                            // 9. محاولة تسجيل الدخول التلقائي
                            authStatusElement.textContent = "Not authenticated. Signing in...";
                            
                            if (initialAuthToken) {
                                try {
                                    await signInWithCustomToken(auth, initialAuthToken);
                                } catch (tokenError) {
                                    console.warn("Custom token failed, trying anonymous", tokenError);
                                    await signInAnonymously(auth);
                                }
                            } else {
                                await signInAnonymously(auth);
                            }
                        }
                    } catch (authError) {
                        // 10. معالجة أخطاء المصادقة بشكل أفضل
                        console.error("Authentication error:", authError);
                        showMessage(`Authentication failed: ${authError.message}`, true);
                        authStatusElement.textContent = "Authentication Error";
                    }
                });

            } catch (initError) {
                console.error("Initialization error:", initError);
                showMessage(`Initialization failed: ${initError.message}`, true);
            }
        };
    </script>
</body>
</html>
