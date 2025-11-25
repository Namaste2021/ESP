<!DOCTYPE html>
<html lang="pt">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Españolízate: Falsos Amigos Español-Português</title>
    <!-- Incluir Tailwind CSS -->
    <script src="https://cdn.tailwindcss.com"></script>
    <!-- Incluir iconos de Lucide (opcional, mas útil) -->
    <script src="https://unpkg.com/lucide@latest"></script>
    <!-- Firebase SDKs -->
    <script type="module">
        import { initializeApp } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-app.js";
        import { getAuth, signInAnonymously, signInWithCustomToken, onAuthStateChanged } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-auth.js";
        // Adicionando imports de Firestore para consultas de ranking
        import { getFirestore, doc, getDoc, setDoc, updateDoc, collection, query, getDocs, setLogLevel } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-firestore.js";

        // Establecer o nível de log para depuração de Firestore
        setLogLevel('debug');

        // Variáveis globais (definidas pelo ambiente Canvas)
        const appId = typeof __app_id !== 'undefined' ? __app_id : 'default-app-id';
        const firebaseConfig = typeof __firebase_config !== 'undefined' ? JSON.parse(__firebase_config) : null;
        const initialAuthToken = typeof __initial_auth_token !== 'undefined' ? __initial_auth_token : null;

        let db;
        let auth;
        let userId = 'anon-user'; // Valor padrão antes da autenticação
        let userHighScore = 0;
        let isAuthReady = false;

        /**
         * Inicializa o Firebase, autentica o usuário e carrega a pontuação máxima.
         */
        window.initFirebase = async function() {
            if (!firebaseConfig) {
                console.error("Firebase config não está disponível. Usando modo local para pontuação.");
                isAuthReady = true;
                document.getElementById('loading-message').textContent = 'Firebase não configurado. Jogando em modo local.';
                window.showIntroScreen();
                return;
            }

            try {
                const app = initializeApp(firebaseConfig);
                db = getFirestore(app);
                auth = getAuth(app);

                // Autenticação usando o token ou anonimamente
                if (initialAuthToken) {
                    await signInWithCustomToken(auth, initialAuthToken);
                } else {
                    await signInAnonymously(auth);
                }

                // Esperar até que o estado de autenticação mude e obter o userId
                onAuthStateChanged(auth, (user) => {
                    if (user) {
                        userId = user.uid;
                        console.log("Usuário autenticado. UID:", userId);
                        // Exibir apenas os primeiros 8 caracteres do ID do usuário para o usuário
                        document.getElementById('user-id-display').textContent = `ID Usuário: ${userId.substring(0, 8)}...`;
                        window.loadHighScore();
                        window.loadGlobalRanking(); // Carregar ranking global
                    } else {
                        console.log("Autenticação anônima ou falhou.");
                        document.getElementById('user-id-display').textContent = 'ID Usuário: Anônimo';
                    }
                    isAuthReady = true;
                    document.getElementById('loading-message').textContent = 'Pronto para jogar!';
                    window.showIntroScreen();
                });

            } catch (error) {
                console.error("Erro ao inicializar Firebase ou autenticar:", error);
                document.getElementById('loading-message').textContent = 'Erro de conexão. Jogando em modo local.';
                isAuthReady = true;
                window.showIntroScreen();
            }
        }

        /**
         * Carrega a pontuação máxima do usuário no Firestore.
         */
        window.loadHighScore = async function() {
            if (!db || !isAuthReady) return;

            // Caminho para dados públicos para pontuações de jogos
            const scoreDocRef = doc(db, `artifacts/${appId}/public/data/cognate_game_scores`, userId);
            try {
                const docSnap = await getDoc(scoreDocRef);
                if (docSnap.exists()) {
                    const data = docSnap.data();
                    userHighScore = data.highScore || 0;
                    userName = data.userName || ''; // Carrega o nome salvo
                    document.getElementById('name-input').value = userName;
                } else {
                    // Inicializar se for a primeira vez
                    await setDoc(scoreDocRef, { highScore: 0, userId: userId, userName: 'Participante' });
                }
                document.getElementById('high-score-display').textContent = userHighScore;
                console.log("Pontuação máxima carregada:", userHighScore);
            } catch (e) {
                console.error("Erro ao carregar pontuação máxima:", e);
            }
        }

        /**
         * Atualiza a pontuação máxima se o score atual for maior.
         */
        window.updateHighScore = async function(currentScore, name) {
            if (!db || !isAuthReady || currentScore <= userHighScore) return;

            userHighScore = currentScore;
            userName = name; // Garante que o nome mais recente seja salvo
            document.getElementById('high-score-display').textContent = userHighScore;
            const scoreDocRef = doc(db, `artifacts/${appId}/public/data/cognate_game_scores`, userId);

            try {
                // Usar setDoc com merge: true para garantir que outros campos sejam mantidos
                await setDoc(scoreDocRef, { 
                    highScore: currentScore, 
                    userId: userId, 
                    userName: name || 'Participante' 
                }, { merge: true });
                console.log("Pontuação máxima atualizada para:", currentScore);
                window.loadGlobalRanking(); // Atualiza o ranking após novo high score
            } catch (e) {
                console.error("Erro ao atualizar pontuação máxima:", e);
            }
        }

        /**
         * Carrega e exibe o ranking global.
         */
        window.loadGlobalRanking = async function() {
            const rankingContainer = document.getElementById('global-ranking-container');
            if (!db || !isAuthReady) return;

            const scoresColRef = collection(db, `artifacts/${appId}/public/data/cognate_game_scores`);
            const q = query(scoresColRef); // Sem orderBy para evitar erro de índice
            
            try {
                const querySnapshot = await getDocs(q);
                let ranking = [];
                querySnapshot.forEach((doc) => {
                    const data = doc.data();
                    if (data.highScore > 0) { // Mostrar apenas jogadores com pontuação
                        ranking.push({
                            name: data.userName || `Usuário ${data.userId.substring(0, 4)}...`,
                            score: data.highScore
                        });
                    }
                });

                // 1. Classificar ranking em JavaScript (descending)
                ranking.sort((a, b) => b.score - a.score);
                // 2. Limitar aos Top 10
                ranking = ranking.slice(0, 10);

                // 3. Renderizar a tabela
                let rankingHtml = `
                    <h3 class="text-2xl font-bold text-indigo-700 mb-4 border-b pb-2">🏆 Ranking Global (Top 10)</h3>
                    <ul class="space-y-2 text-left">
                `;

                if (ranking.length === 0) {
                    rankingHtml += `<li class="text-gray-500 italic">Seja o primeiro a pontuar!</li>`;
                } else {
                    ranking.forEach((entry, index) => {
                        const rank = index + 1;
                        const bgColor = rank === 1 ? 'bg-yellow-100 border-yellow-500' : 
                                        rank === 2 ? 'bg-gray-100 border-gray-400' : 
                                        rank === 3 ? 'bg-orange-100 border-orange-400' : 
                                        'bg-white border-indigo-100';
                        const textStyle = rank <= 3 ? 'font-extrabold' : 'font-medium';

                        rankingHtml += `
                            <li class="flex justify-between items-center p-3 rounded-lg border-l-4 ${bgColor} shadow-sm">
                                <span class="text-lg ${textStyle} text-indigo-800">${rank}. ${entry.name}</span>
                                <span class="text-xl ${textStyle} text-indigo-600">${entry.score} pts</span>
                            </li>
                        `;
                    });
                }
                
                rankingHtml += `</ul>`;
                rankingContainer.innerHTML = rankingHtml;

            } catch (e) {
                console.error("Erro ao carregar ranking global:", e);
                rankingContainer.innerHTML = `<p class="text-red-500 mt-4">Erro ao carregar ranking.</p>`;
            }
        }

        // Iniciar a inicialização do Firebase ao carregar a janela
        window.onload = window.initFirebase;

    </script>
    <style>
        @import url('https://fonts.googleapis.com/css2?family=Inter:wght@400;600;800&display=swap');
        body {
            font-family: 'Inter', sans-serif;
            background-color: #f7f7f7;
            color: #1f2937;
        }
        .quiz-container {
            transition: all 0.5s ease-in-out;
            background-size: cover;
            background-position: center;
            min-height: 500px;
            display: flex;
            flex-direction: column;
            justify-content: space-between;
        }
        .option-button {
            transition: all 0.2s;
            box-shadow: 0 4px 6px -1px rgba(0, 0, 0, 0.1), 0 2px 4px -2px rgba(0, 0, 0, 0.1);
        }
        .option-button:hover {
            transform: translateY(-2px);
            box-shadow: 0 10px 15px -3px rgba(0, 0, 0, 0.1), 0 4px 6px -4px rgba(0, 0, 0, 0.1);
        }
        .correct {
            background-color: #10b981 !important; /* Green 500 */
            color: white !important;
        }
        .incorrect {
            background-color: #ef4444 !important; /* Red 500 */
            color: white !important;
        }

        /* --- CSS para a Animação Festiva (Confeti) --- */
        .confetti-container {
            position: absolute;
            top: 0;
            left: 0;
            width: 100%;
            height: 100%;
            pointer-events: none;
            overflow: hidden;
        }
        .confetti {
            position: absolute;
            width: 10px;
            height: 10px;
            background-color: #f00; /* Cor inicial, é sobrescrita no JS */
            opacity: 0;
            animation: fall 3s ease-in forwards;
        }
        @keyframes fall {
            0% { transform: translateY(-10vh) rotate(0deg); opacity: 1; }
            100% { transform: translateY(100vh) rotate(720deg); opacity: 0; }
        }
        /* Nova animação para a tela de carregamento do nome */
        .dot-pulse {
            width: 10px;
            height: 10px;
            border-radius: 50%;
            background-color: #4f46e5;
            animation: dot-pulse-anim 1s infinite alternate;
            margin: 0 4px;
        }

        @keyframes dot-pulse-anim {
            0% { transform: scale(1); opacity: 0.7; }
            100% { transform: scale(1.5); opacity: 1; }
        }
    </style>
</head>
<body class="p-4 md:p-8 min-h-screen">

    <!-- Mensagens de status -->
    <div id="status-bar" class="flex flex-col md:flex-row justify-between items-center mb-6 p-4 bg-white rounded-xl shadow-lg border-b-4 border-indigo-500">
        <h1 class="text-3xl font-extrabold text-indigo-700 mb-2 md:mb-0">
            Españolízate
        </h1>
        <div class="flex items-center space-x-4 text-sm font-medium text-gray-600">
            <span id="user-id-display" class="bg-gray-100 p-2 rounded-lg">Carregando usuário...</span>
            <span>Pontuação Máxima: <span id="high-score-display" class="font-bold text-lg text-indigo-600">0</span></span>
        </div>
    </div>

    <!-- Tela de Carregamento -->
    <div id="loading-screen" class="text-center py-20">
        <div class="animate-spin rounded-full h-12 w-12 border-b-2 border-indigo-500 mx-auto mb-4"></div>
        <p id="loading-message" class="text-gray-600 font-semibold">Iniciando Firebase e autenticando...</p>
    </div>

    <div id="game-content" class="max-w-4xl mx-auto hidden">
        
        <!-- TELA DE INTRODUÇÃO (FALSOS AMIGOS) -->
        <div id="intro-screen" class="bg-white rounded-3xl shadow-2xl p-8 md:p-12 transition-all duration-500 border-8 border-indigo-200 grid grid-cols-1 lg:grid-cols-3 gap-8">
            
            <!-- Coluna Principal (Jogo e Nome) -->
            <div class="lg:col-span-2">
                <div class="text-center">
                    <i data-lucide="globe" class="w-16 h-16 text-indigo-500 mx-auto mb-4 animate-bounce"></i>
                    <h2 class="text-4xl font-extrabold text-indigo-700 mb-4">¡Cuidado con los Falsos Amigos!</h2>
                    
                    <p class="text-lg text-gray-700 mb-6">
                        Aprender espanhol é fácil porque muitas palavras se parecem ao português (são cognatas!). Mas tenha muito cuidado... algumas palavras são enganosas!
                    </p>
                    
                    <!-- Campo de Nome (Movido para cima) -->
                    <div class="mb-8 pt-4 border-b border-indigo-300">
                        <label for="name-input" class="block text-lg font-semibold text-indigo-800 mb-2 text-left">Digite seu nome:</label>
                        <div class="flex flex-col sm:flex-row gap-2">
                            <input type="text" id="name-input" placeholder="Seu Nome Aqui" class="flex-grow p-3 border border-gray-300 rounded-lg focus:ring-indigo-500 focus:border-indigo-500 shadow-inner" maxlength="30">
                            <button id="save-name-button" class="py-3 px-6 bg-green-500 text-white font-bold rounded-lg shadow-md hover:bg-green-600 transition-colors" onclick="saveNameAndFetchInfo()">
                                Guardar Nome e Info ✅
                            </button>
                        </div>
                        <!-- Display para o significado do nome -->
                        <div id="name-info-display" class="mt-4 hidden text-sm text-left">
                            <!-- Conteúdo dinâmico do dado curioso ou animação de carregamento -->
                        </div>
                    </div>

                    <div class="bg-indigo-50 p-6 rounded-xl border-l-4 border-indigo-500 mb-8 text-left">
                        <h3 class="text-xl font-bold text-indigo-800 mb-2">O que são Falsos Cognatos?</h3>
                        <p class="text-gray-600 italic">
                            São palavras que se escrevem ou soam muito similar em dois idiomas (espanhol e português), mas têm significados totalmente distintos. São "falsos amigos" que fazem você acreditar em algo que não é!
                        </p>
                        <p class="mt-4 font-semibold text-sm text-indigo-600">
                            Sua missão neste jogo é descobrir o significado real em português da palavra em espanhol destacada na frase.
                        </p>
                    </div>

                    <div class="flex justify-center items-center space-x-8 text-xl font-bold mb-8">
                        <span class="text-green-600">Acertar = +10 Pontos!</span>
                        <i data-lucide="check-circle" class="w-6 h-6 text-green-500"></i>
                        <i data-lucide="x-circle" class="w-6 h-6 text-red-500"></i>
                        <span class="text-red-600">Errar = -5 Pontos!</span>
                    </div>
                    
                    <!-- Botão de Jugar ativo em todo momento -->
                    <button class="w-full py-4 bg-indigo-600 text-white font-bold text-xl rounded-xl shadow-lg hover:bg-indigo-700 option-button transition-transform duration-300 transform hover:scale-105" onclick="startGame()">
                        <i data-lucide="play-circle" class="inline-block mr-2 w-6 h-6"></i>
                        Começar a Jogar!
                    </button>
                </div>
            </div>

            <!-- Coluna de Ranking Global -->
            <div class="lg:col-span-1 bg-gray-50 p-6 rounded-2xl shadow-inner border border-gray-200" id="global-ranking-container">
                <div class="text-center text-gray-500">
                    <div class="animate-pulse">Carregando Ranking...</div>
                </div>
            </div>

        </div>

        <!-- Contêiner Principal do Jogo (Inicialmente oculto) -->
        <div id="quiz-container" class="quiz-container bg-indigo-100 rounded-3xl shadow-2xl p-6 md:p-10 transition-all duration-500 border-8 border-white hidden">

            <!-- Cabeçalho e Progresso -->
            <div class="flex justify-between items-center mb-8">
                <span class="text-2xl font-bold text-indigo-800 bg-white px-4 py-2 rounded-full shadow-md">Pontos: <span id="score-display">0</span></span>
                <span id="progress-display" class="text-lg font-semibold text-gray-700">Pergunta 1/10</span>
            </div>

            <!-- Área da Pergunta -->
            <div class="text-center bg-white/80 backdrop-blur-sm p-6 md:p-8 rounded-2xl shadow-xl">
                <p class="text-3xl md:text-5xl font-extrabold text-gray-800 leading-snug" id="question-text">
                    La manga está amarilla.
                </p>
                <p class="mt-4 text-lg font-medium text-indigo-600 italic">
                    O que significa a palavra destacada neste contexto?
                </p>
            </div>

            <!-- Área de Opções (em Português) -->
            <div class="mt-10 grid grid-cols-1 md:grid-cols-3 gap-4" id="options-container">
                <!-- Botões de opções gerados por JS -->
            </div>

            <!-- Mensagem de Feedback -->
            <div id="feedback-message" class="mt-6 text-center text-xl font-bold rounded-xl p-3 hidden"></div>

            <!-- Botão de Próximo -->
            <button id="next-button" class="mt-6 w-full py-3 bg-indigo-600 text-white font-bold text-lg rounded-xl shadow-lg hover:bg-indigo-700 option-button" onclick="nextQuestion()" disabled>
                Próxima Pergunta
                <i data-lucide="arrow-right" class="inline-block ml-2 w-5 h-5"></i>
            </button>
        </div>

        <!-- Tela Final (Modal) -->
        <div id="result-modal" class="fixed inset-0 bg-gray-900 bg-opacity-70 flex items-center justify-center p-4 hidden z-50">
            <div class="confetti-container" id="confetti-container"></div> <!-- Contêiner para a animação de confete -->
            <div class="bg-white rounded-xl shadow-2xl p-8 md:p-10 max-w-lg w-full text-center transform transition-all duration-300 scale-95 relative z-10">
                <i data-lucide="trophy" class="w-16 h-16 text-yellow-500 mx-auto mb-4"></i>
                <h2 id="modal-title" class="text-3xl font-extrabold text-indigo-700 mb-3">Jogo Terminado</h2>
                <p class="text-xl text-gray-700 mb-2">Sua Pontuação Final é:</p>
                <p class="text-4xl text-gray-900 font-black mb-4">
                    <span id="final-score" class="text-indigo-600">0</span> / 100
                </p>
                <p id="new-high-score-message" class="text-green-600 font-bold mb-6 hidden">Parabéns, você estabeleceu um novo recorde!</p>
                <p class="text-sm text-gray-500 mb-6">Sua pontuação máxima salva é: <span id="modal-high-score">0</span></p>

                <button class="w-full py-3 bg-indigo-600 text-white font-bold text-lg rounded-xl shadow-lg hover:bg-indigo-700 option-button" onclick="window.location.reload()">
                    <i data-lucide="rotate-ccw" class="inline-block mr-2 w-5 h-5"></i>
                    Jogar Novamente
                </button>
            </div>
        </div>
    </div>

    <script>
        // --- DADOS DE PERGUNTAS (ESPANHOL vs. PORTUGUÊS) ---
        const questions = [
            {
                id: 1,
                sentence: "La manga está amarilla.",
                word: "manga",
                correct: "a fruta",
                incorrect_polisemia: "a parte da camisa",
                incorrect_falso: "a mão",
                answer: "a fruta",
                image_url: "https://placehold.co/800x600/10b981/ffffff?text=MANGA+FRESCA"
            },
            {
                id: 2,
                sentence: "Compré una taza de café.",
                word: "taza",
                correct: "o copo para bebidas quentes",
                incorrect_polisemia: "o cálice / o troféu",
                incorrect_falso: "o prato",
                answer: "o copo para bebidas quentes",
                image_url: "https://placehold.co/800x600/3b82f6/ffffff?text=TAZA+DE+CAFE"
            },
            {
                id: 3,
                sentence: "La casa tiene un balcón con vista al mar.",
                word: "balcón",
                correct: "a sacada",
                incorrect_polisemia: "o balcão de uma loja",
                incorrect_falso: "o andaime",
                answer: "a sacada",
                image_url: "https://placehold.co/800x600/6366f1/ffffff?text=BALCON+VISTA+AL+MAR"
            },
            {
                id: 4,
                sentence: "Este postre está realmente exquisito.",
                word: "exquisito",
                correct: "delicioso / saboroso",
                incorrect_polisemia: "estranho / bizarro",
                incorrect_falso: "caro / refinado",
                answer: "delicioso / saboroso",
                image_url: "https://placehold.co/800x600/ef4444/ffffff?text=POSTRE+DELICIOSO"
            },
            {
                id: 5,
                sentence: "Borra tu nombre de la lista, por favor.",
                word: "Borra",
                correct: "apagar / eliminar",
                incorrect_polisemia: "fazer uma bagunça",
                incorrect_falso: "furar / fazer buracos",
                answer: "apagar / eliminar",
                image_url: "https://placehold.co/800x600/f59e0b/ffffff?text=BORRAR+LISTA"
            },
            {
                id: 6,
                sentence: "Ella tiene mucha gracia al bailar.",
                word: "gracia",
                correct: "elegância / charme",
                incorrect_polisemia: "favor / benefício",
                incorrect_falso: "gordura / óleo",
                answer: "elegância / charme",
                image_url: "https://placehold.co/800x600/ec4899/ffffff?text=BAILAR+CON+GRACIA"
            },
            {
                id: 7,
                sentence: "Necesito un apellido para mi visa.",
                word: "apellido",
                correct: "o sobrenome",
                incorrect_polisemia: "o cognome / apelido",
                incorrect_falso: "o chamado / a solicitação",
                answer: "o sobrenome",
                image_url: "https://placehold.co/800x600/14b8a6/ffffff?text=APELLIDO+FAMILIAR"
            },
            {
                id: 8,
                sentence: "El camarero trajo el vaso de agua.",
                word: "vaso",
                correct: "o copo",
                incorrect_polisemia: "o jarro / a jarra",
                incorrect_falso: "o vaso de planta",
                answer: "o copo",
                image_url: "https://placehold.co/800x600/a855f7/ffffff?text=VASO+DE+AGUA"
            },
            {
                id: 9,
                sentence: "Debes contestar la pregunta.",
                word: "contestar",
                correct: "responder / replicar",
                incorrect_polisemia: "protestar / contradizer",
                incorrect_falso: "dar um sermão",
                answer: "responder / replicar",
                image_url: "https://placehold.co/800x600/f97316/ffffff?text=CONTESTAR+PREGUNTA"
            },
            {
                id: 10,
                sentence: "El joven tiene una oficina en el centro.",
                word: "oficina",
                correct: "o escritório",
                incorrect_polisemia: "o ofício / a profissão",
                incorrect_falso: "a oficina mecânica",
                answer: "o escritório",
                image_url: "https://placehold.co/800x600/06b6d4/ffffff?text=OFICINA+ESCRITORIO"
            }
        ];

        // --- ESTADO DO JOGO ---
        let currentQuestionIndex = 0;
        let score = 0;
        let isQuestionActive = true;
        let shuffledQuestions = [];
        const TOTAL_QUESTIONS = 10;
        let userName = ''; // Armazena o nome do usuário

        /**
         * Exibe a tela de introdução uma vez que o Firebase está pronto.
         */
        window.showIntroScreen = function() {
            document.getElementById('loading-screen').classList.add('hidden');
            document.getElementById('game-content').classList.remove('hidden');
            document.getElementById('intro-screen').classList.remove('hidden');
            document.getElementById('quiz-container').classList.add('hidden');
            lucide.createIcons(); // Inicializar ícones de Lucide
        }

        /**
         * Mistura as perguntas e começa o jogo (ao pressionar o botão de início).
         */
        window.startGame = function() {
            // Se o usuário não salvou um nome, usamos o que estiver no input ou um valor padrão.
            // O nome é garantido para ser capitalizado e armazenado na variável global 'userName'
            if (!userName) {
                 userName = document.getElementById('name-input').value.trim() || 'Participante';
                // Certifica-se de que o nome está capitalizado para exibição
                userName = userName.charAt(0).toUpperCase() + userName.slice(1).toLowerCase();
            }

            // Ocultar intro e exibir quiz
            document.getElementById('intro-screen').classList.add('hidden');
            document.getElementById('quiz-container').classList.remove('hidden');

            // Misturar perguntas para variedade
            shuffledQuestions = questions.sort(() => Math.random() - 0.5).slice(0, TOTAL_QUESTIONS);
            currentQuestionIndex = 0;
            score = 0;
            updateScoreDisplay();
            loadQuestion();
        }

        /**
         * Atualiza a pontuação e o progresso na interface.
         */
        function updateScoreDisplay() {
            document.getElementById('score-display').textContent = score;
            document.getElementById('progress-display').textContent = `Pergunta ${currentQuestionIndex + 1}/${TOTAL_QUESTIONS}`;
        }

        /**
         * Cria e anima partículas de confete.
         */
        function launchConfetti() {
            const container = document.getElementById('confetti-container');
            container.innerHTML = ''; // Limpar confeti anterior
            const colors = ['#f44336', '#e91e63', '#9c27b0', '#673ab7', '#3f51b5', '#2196f3', '#03a9f4', '#00bcd4', '#009688', '#4caf50', '#8bc34a', '#cddc39', '#ffeb3b', '#ffc107', '#ff9800', '#ff5722'];

            for (let i = 0; i < 50; i++) {
                const confetti = document.createElement('div');
                confetti.className = 'confetti';
                confetti.style.backgroundColor = colors[Math.floor(Math.random() * colors.length)];
                confetti.style.left = `${Math.random() * 100}vw`; // Posição inicial aleatória (viewport width)
                confetti.style.width = `${Math.random() * 5 + 5}px`;
                confetti.style.height = `${Math.random() * 5 + 5}px`;
                confetti.style.animationDelay = `${Math.random() * 2}s`;
                confetti.style.animationDuration = `${3 + Math.random() * 2}s`; // Duração aleatória para variar a queda
                
                container.appendChild(confetti);
            }
        }

        /**
         * Carrega a pergunta atual na interface.
         */
        function loadQuestion() {
            if (currentQuestionIndex >= shuffledQuestions.length) {
                showResultModal();
                return;
            }

            const q = shuffledQuestions[currentQuestionIndex];
            const container = document.getElementById('quiz-container');
            const optionsContainer = document.getElementById('options-container');

            // 1. Exibir texto da pergunta (destacando a palavra dinamicamente em espanhol)
            // Uso de uma expressão regular para lidar com maiúsculas/minúsculas no destaque
            const wordRegex = new RegExp(q.word, 'gi'); 
            const formattedSentence = q.sentence.replace(wordRegex, (match) => 
                `<span class="text-indigo-600 underline decoration-4 decoration-red-400 font-bold">${match}</span>`
            );
            document.getElementById('question-text').innerHTML = formattedSentence;

            // 2. Definir a imagem de fundo
            container.style.backgroundImage = `linear-gradient(rgba(255, 255, 255, 0.7), rgba(255, 255, 255, 0.9)), url('${q.image_url}')`;

            // 3. Preparar opções
            const allOptions = [
                { text: q.correct, isCorrect: true },
                { text: q.incorrect_polisemia, isCorrect: false },
                { text: q.incorrect_falso, isCorrect: false }
            ].sort(() => Math.random() - 0.5); // Misturar opções

            // 4. Renderizar opções
            optionsContainer.innerHTML = '';
            allOptions.forEach(option => {
                const button = document.createElement('button');
                button.textContent = option.text;
                button.className = 'option-button py-3 px-4 bg-white text-gray-800 font-semibold rounded-xl text-center border-2 border-indigo-400 hover:bg-indigo-50 shadow-md';
                button.onclick = () => checkAnswer(button, option.isCorrect);
                optionsContainer.appendChild(button);
            });

            // Resetar estado
            isQuestionActive = true;
            document.getElementById('feedback-message').classList.add('hidden');
            document.getElementById('next-button').disabled = true;
        }

        /**
         * Verifica a resposta do usuário.
         * @param {HTMLElement} button - O botão clicado.
         * @param {boolean} isCorrect - Se a resposta está correta.
         */
        function checkAnswer(button, isCorrect) {
            if (!isQuestionActive) return;

            isQuestionActive = false;
            const feedback = document.getElementById('feedback-message');
            const nextButton = document.getElementById('next-button');
            const options = document.getElementById('options-container').children;

            if (isCorrect) {
                score += 10;
                feedback.textContent = 'Correto! +10 Pontos.';
                feedback.className = 'mt-6 text-center text-xl font-bold rounded-xl p-3 bg-green-100 text-green-700';
                button.classList.add('correct');
            } else {
                score = Math.max(0, score - 5); // Penalidade leve
                feedback.textContent = 'Incorreto. Revise o significado no contexto! (-5 Pontos)';
                feedback.className = 'mt-6 text-center text-xl font-bold rounded-xl p-3 bg-red-100 text-red-700';
                button.classList.add('incorrect');
            }

            // Destacar a resposta correta e desativar todos os botões
            for (const opt of options) {
                const q = shuffledQuestions[currentQuestionIndex];
                if (opt.textContent.trim() === q.answer.trim()) {
                    // Sempre destacar a correta, mesmo se o usuário acertou
                    if (!isCorrect) {
                        opt.classList.add('correct');
                    }
                }
                opt.disabled = true; 
                opt.onclick = null;
            }
            
            updateScoreDisplay();
            feedback.classList.remove('hidden');
            nextButton.disabled = false;
        }

        /**
         * Avança para a próxima pergunta.
         */
        window.nextQuestion = function() {
            currentQuestionIndex++;
            document.getElementById('next-button').disabled = true;
            if (currentQuestionIndex < shuffledQuestions.length) {
                loadQuestion();
            } else {
                showResultModal();
            }
        }

        /**
         * Exibe o modal de resultados e atualiza a pontuação máxima.
         */
        function showResultModal() {
            const modal = document.getElementById('result-modal');
            const finalScoreDisplay = document.getElementById('final-score');
            const modalTitle = document.getElementById('modal-title');
            const newHighScoreMessage = document.getElementById('new-high-score-message');
            const modalHighScoreDisplay = document.getElementById('modal-high-score');
            
            finalScoreDisplay.textContent = score;
            
            // Verificar e atualizar a pontuação máxima
            const isNewHighScore = score > userHighScore;
            if (isNewHighScore) {
                window.updateHighScore(score, userName); // Passa o nome para salvar junto com a pontuação
                newHighScoreMessage.classList.remove('hidden');
                modalHighScoreDisplay.textContent = score; // Atualiza o modal com o novo score
            } else {
                newHighScoreMessage.classList.add('hidden');
                modalHighScoreDisplay.textContent = userHighScore; // Mantém o score anterior
            }

            // Personalizar o título com o nome
            const displayUserName = userName || 'Campeão';
            modalTitle.textContent = `¡Felicidades, ${displayUserName}!`;
            
            // Mostrar confeti se a pontuação for boa (>= 70)
            if (score >= 70) {
                launchConfetti();
            }
            
            modal.classList.remove('hidden');
        }
        
        // --- FUNÇÕES PARA INTEGRAÇÃO COM GEMINI API ---

        const apiKey = ""; // API key is fornecida pelo ambiente

        async function fetchWithRetry(url, options, maxRetries = 5) {
            for (let i = 0; i < maxRetries; i++) {
                try {
                    const response = await fetch(url, options);
                    if (!response.ok) throw new Error(`HTTP error! status: ${response.status}`);
                    return response;
                } catch (error) {
                    if (i === maxRetries - 1) throw error;
                    const delay = Math.pow(2, i) * 1000;
                    console.warn(`Fetch attempt ${i + 1} failed. Retrying in ${delay / 1000}s...`, error);
                    await new Promise(resolve => setTimeout(resolve, delay));
                }
            }
        }

        /**
         * Busca o significado e dado curioso do nome usando a API.
         */
        window.fetchNameInfo = async function(name) {
            const infoDisplay = document.getElementById('name-info-display');
            const nameInput = document.getElementById('name-input');
            const saveButton = document.getElementById('save-name-button');

            // Animação criativa sem mensagem explícita de busca
            infoDisplay.innerHTML = `
                <div class="text-indigo-600 font-semibold flex items-center justify-center p-2">
                    <div class="dot-pulse"></div>
                    <div class="dot-pulse" style="animation-delay: 0.2s;"></div>
                    <div class="dot-pulse" style="animation-delay: 0.4s;"></div>
                </div>
            `;
            infoDisplay.classList.remove('hidden');
            lucide.createIcons();
            
            // Bloquear input e botão durante a busca
            nameInput.disabled = true;
            saveButton.disabled = true;
            
            // System prompt ajustado para obter um dado curioso em formato de 'Sabías que...'
            const systemPrompt = "Actúa como un narrador de datos curiosos. Proporciona un dato curioso sobre el origen o significado del nombre dado en español, comenzando siempre con la frase: 'Sabías que...' y usando no más de 100 palabras en total.";
            const userQuery = `Dime un dato curioso, su origen y significado sobre el nombre ${name}.`;
            
            const apiUrl = `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash-preview-09-2025:generateContent?key=${apiKey}`;

            const payload = {
                contents: [{ parts: [{ text: userQuery }] }],
                tools: [{ "google_search": {} }],
                systemInstruction: {
                    parts: [{ text: systemPrompt }]
                },
            };

            try {
                const response = await fetchWithRetry(apiUrl, {
                    method: 'POST',
                    headers: { 'Content-Type': 'application/json' },
                    body: JSON.stringify(payload)
                });
                const result = await response.json();
                
                const candidate = result.candidates?.[0];
                let generatedText = "No se pudo encontrar información detallada sobre tu nombre.";
                let sources = [];

                if (candidate && candidate.content?.parts?.[0]?.text) {
                    generatedText = candidate.content.parts[0].text;
                    
                    // Extrair fontes de fundamentação
                    const groundingMetadata = candidate.groundingMetadata;
                    if (groundingMetadata && groundingMetadata.groundingAttributions) {
                        sources = groundingMetadata.groundingAttributions
                            .map(attribution => ({
                                uri: attribution.web?.uri,
                                title: attribution.web?.title,
                            }))
                            .filter(source => source.uri && source.title); // Garantir que as fontes são válidas
                    }
                }

                // Formatar o output
                let sourcesHtml = sources.length > 0
                    ? `<p class="mt-2 text-xs text-gray-500">Fontes: ${sources.map(s => `<a href="${s.uri}" target="_blank" class="underline hover:text-indigo-500">${s.title}</a>`).join(', ')}</p>`
                    : '';

                infoDisplay.innerHTML = `
                    <div class="bg-white p-3 rounded-lg border border-indigo-300 shadow-inner">
                        <p class="text-sm text-gray-700 font-medium italic">${generatedText}</p>
                        ${sourcesHtml}
                    </div>
                `;

            } catch (error) {
                console.error("Erro ao buscar informações do nome:", error);
                infoDisplay.innerHTML = `<p class="text-red-500">Erro ao carregar dado curioso. Tente novamente.</p>`;
                // Reativar em caso de erro para permitir nova tentativa
                nameInput.disabled = false;
                saveButton.disabled = false;
            }
        }

        /**
         * Função de controle: salva o nome e dispara a busca de informações.
         */
        window.saveNameAndFetchInfo = function() {
            const input = document.getElementById('name-input');
            const name = input.value.trim();

            if (name) {
                // Capitalizar e armazenar o nome
                userName = name.charAt(0).toUpperCase() + name.slice(1).toLowerCase();
                
                // Salvar o nome no Firestore (mesmo que a pontuação seja 0)
                if (window.updateHighScore) {
                    window.updateHighScore(userHighScore, userName); 
                }
                
                fetchNameInfo(userName);
            } else {
                // Se o campo estiver vazio, apenas definimos um nome padrão e limpamos a área de info
                userName = 'Participante';
                document.getElementById('name-info-display').classList.add('hidden');
                // Reativar input e botão se o usuário limpar o campo
                input.disabled = false;
                document.getElementById('save-name-button').disabled = false;
            }
        }
    </script>
</body>
</html>
