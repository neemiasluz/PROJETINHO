import React, { useState, useEffect, useCallback, useMemo } from 'react';
import { initializeApp } from 'firebase/app';
import { getAuth, signInAnonymously, signInWithCustomToken, onAuthStateChanged } from 'firebase/auth';
import { getFirestore, doc, setDoc, collection, query, orderBy, onSnapshot, serverTimestamp, getDocs, addDoc } from 'firebase/firestore';

// Variáveis globais de ambiente fornecidas pelo Canvas
const appId = typeof __app_id !== 'undefined' ? __app_id : 'default-app-id';
const firebaseConfig = typeof __firebase_config !== 'undefined' ? JSON.parse(__firebase_config) : {};
const initialAuthToken = typeof __initial_auth_token !== 'undefined' ? __initial_auth_token : undefined;
const API_KEY = ""; // Chave da API para Gemini

// --- UTILITIES E CONFIGURAÇÃO DE IA ---

/**
 * Função utilitária para chamar a API do Gemini com retentativas (exponential backoff).
 * @param {string} userQuery - A consulta específica do usuário.
 * @param {string} systemPrompt - O papel e as regras da IA.
 * @param {boolean} useGrounding - Se deve usar a Pesquisa Google para fundamentação.
 * @returns {Promise<{text: string, sources: Array<{uri: string, title: string}>}>}
 */
const callGeminiApi = async (userQuery, systemPrompt, useGrounding = true) => {
    const model = 'gemini-2.5-flash-preview-09-2025';
    const apiUrl = `https://generativelanguage.googleapis.com/v1beta/models/${model}:generateContent?key=${API_KEY}`;
    
    const payload = {
        contents: [{ parts: [{ text: userQuery }] }],
        systemInstruction: { parts: [{ text: systemPrompt }] },
    };

    if (useGrounding) {
        payload.tools = [{ "google_search": {} }];
    }

    const MAX_RETRIES = 3;
    let delay = 1000;

    for (let i = 0; i < MAX_RETRIES; i++) {
        try {
            const response = await fetch(apiUrl, {
                method: 'POST',
                headers: { 'Content-Type': 'application/json' },
                body: JSON.stringify(payload)
            });

            if (response.status === 429 && i < MAX_RETRIES - 1) {
                // Too Many Requests - Wait and retry
                await new Promise(resolve => setTimeout(resolve, delay));
                delay *= 2; // Exponential backoff
                continue;
            }

            const result = await response.json();
            const candidate = result.candidates?.[0];

            if (candidate && candidate.content?.parts?.[0]?.text) {
                const text = candidate.content.parts[0].text;
                let sources = [];
                const groundingMetadata = candidate.groundingMetadata;

                if (groundingMetadata && groundingMetadata.groundingAttributions) {
                    sources = groundingMetadata.groundingAttributions
                        .map(attribution => ({
                            uri: attribution.web?.uri,
                            title: attribution.web?.title,
                        }))
                        .filter(source => source.uri && source.title);
                }
                return { text, sources };
            }

            throw new Error(result.error?.message || "Resposta da IA vazia ou mal formatada.");

        } catch (error) {
            console.error(`Tentativa ${i + 1} falhou:`, error);
            if (i === MAX_RETRIES - 1) throw error;
            await new Promise(resolve => setTimeout(resolve, delay));
            delay *= 2;
        }
    }
    return { text: "Desculpe, a IA não conseguiu gerar a resposta no momento. Tente novamente mais tarde.", sources: [] };
};

// --- MOCK DATA SIMULADO (AUMENTADO PARA 25 QUESTÕES) ---
const DISCIPLINES = [
    'Língua Portuguesa', 'História de Pernambuco', 'Raciocínio Lógico', 
    'Informática', 'Direitos e Garantias Fundamentais'
];
const MOCK_QUESTIONS = [
    {
        id: 1, topic: 'Interpretação de Texto', discipline: 'Língua Portuguesa', difficulty: 'Média', year: 2023,
        prompt: "Assinale a alternativa que apresenta a principal função da crase na língua portuguesa, considerando o uso formal.",
        options: ["Marcar o plural.", "Indicar a elisão do 'a' preposição com o 'a' artigo ou pronome.", "Substituir a vírgula.", "Enfatizar o substantivo."],
        answer: "Indicar a elisão do 'a' preposição com o 'a' artigo ou pronome.",
    },
    {
        id: 2, topic: 'Invasões Holandesas', discipline: 'História de Pernambuco', difficulty: 'Fácil', year: 2022,
        prompt: "Qual foi o principal objetivo da Companhia das Índias Ocidentais (WIC) ao invadir Pernambuco no século XVII?",
        options: ["Estabelecer uma colônia de povoamento.", "Controlar a produção de ouro.", "Assumir o controle da produção açucareira.", "Fomentar o comércio de especiarias."],
        answer: "Assumir o controle da produção açucareira.",
    },
    {
        id: 3, topic: 'Diagramas Lógicos', discipline: 'Raciocínio Lógico', difficulty: 'Difícil', year: 2024,
        prompt: "Se 'Todo policial é dedicado' e 'Nenhum dedicado é corrupto', qual a conclusão lógica correta?",
        options: ["Algum corrupto é policial.", "Todo corrupto é não dedicado.", "Nenhum policial é corrupto.", "Algum policial é corrupto."],
        answer: "Nenhum policial é corrupto.",
    },
    {
        id: 4, topic: 'Segurança da Informação', discipline: 'Informática', difficulty: 'Média', year: 2023,
        prompt: "Um ataque de 'Phishing' é caracterizado por:",
        options: ["Ataque de negação de serviço (DoS).", "Tentativa de obter dados sensíveis (senhas, cartões) por meio de comunicação falsa.", "Instalação de hardware malicioso.", "Criptografia de arquivos para resgate."],
        answer: "Tentativa de obter dados sensíveis (senhas, cartões) por meio de comunicação falsa.",
    },
    {
        id: 5, topic: 'Direitos Fundamentais', discipline: 'Direitos e Garantias Fundamentais', difficulty: 'Fácil', year: 2024,
        prompt: "O direito à vida, à liberdade, à igualdade, à segurança e à propriedade, conforme a Constituição Federal de 1988, é classificado como:",
        options: ["Direitos Sociais.", "Direitos Políticos.", "Direitos Individuais e Coletivos.", "Direitos de Segunda Geração."],
        answer: "Direitos Individuais e Coletivos.",
    },
    {
        id: 6, topic: 'Concordância Verbal', discipline: 'Língua Portuguesa', difficulty: 'Média', year: 2022,
        prompt: "Assinale a frase com erro de concordância verbal:",
        options: ["Faz dois anos que ele estuda.", "Havia muitos candidatos na sala.", "Vão fazer 10 dias do edital.", "Choveu forte na capital."],
        answer: "Vão fazer 10 dias do edital.",
    },
    {
        id: 7, topic: 'Revolução Praieira', discipline: 'História de Pernambuco', difficulty: 'Difícil', year: 2024,
        prompt: "O principal lema da Revolução Praieira (1848) era:",
        options: ["Liberdade, Igualdade e Fraternidade.", "Comércio livre, voto aberto e federalismo.", "Guerra aos cabanos e fim da escravidão.", "Reforma agrária e poder centralizado."],
        answer: "Comércio livre, voto aberto e federalismo.",
    },
    // Novas 18 questões adicionadas:
    {
        id: 8, topic: 'Regência Verbal', discipline: 'Língua Portuguesa', difficulty: 'Média', year: 2024,
        prompt: "Em qual das frases a regência do verbo 'aspirar' está corretamente empregada?",
        options: ["Ele aspirava o cargo de comandante.", "Ele aspirava ao cargo de comandante.", "Ele aspirava um ar puro.", "Ele aspirou por um aumento."],
        answer: "Ele aspirava ao cargo de comandante.",
    },
    {
        id: 9, topic: 'Coesão e Coerência', discipline: 'Língua Portuguesa', difficulty: 'Difícil', year: 2023,
        prompt: "O termo coesão referencial é um mecanismo que se utiliza para:",
        options: ["Garantir a progressão do texto.", "Evitar repetições desnecessárias, remetendo a algo já mencionado.", "Estabelecer relações lógicas entre orações.", "Organizar parágrafos por ordem cronológica."],
        answer: "Evitar repetições desnecessárias, remetendo a algo já mencionado.",
    },
    {
        id: 10, topic: 'Pontuação', discipline: 'Língua Portuguesa', difficulty: 'Fácil', year: 2022,
        prompt: "Em qual caso a vírgula é obrigatória?",
        options: ["Entre sujeito e predicado.", "Separando elementos de uma enumeração.", "Antes da conjunção 'e' ligando orações coordenadas aditivas com o mesmo sujeito.", "Entre o verbo e seu objeto direto."],
        answer: "Separando elementos de uma enumeração.",
    },
    {
        id: 11, topic: 'Revolução dos Mascates', discipline: 'História de Pernambuco', difficulty: 'Média', year: 1710,
        prompt: "A Revolução dos Mascates (1710-1711) foi um conflito entre quais cidades/grupos?",
        options: ["Olinda (senhores de engenho) e Salvador (comerciantes).", "Recife (comerciantes) e Olinda (senhores de engenho).", "Recife (holandeses) e Olinda (portugueses).", "Igarassu e Cabo de Santo Agostinho."],
        answer: "Recife (comerciantes) e Olinda (senhores de engenho).",
    },
    {
        id: 12, topic: 'Colonização', discipline: 'História de Pernambuco', difficulty: 'Fácil', year: 2021,
        prompt: "Qual produto foi a base da economia de Pernambuco durante o ciclo colonial?",
        options: ["Ouro.", "Algodão.", "Pau-brasil.", "Cana-de-açúcar."],
        answer: "Cana-de-açúcar.",
    },
    {
        id: 13, topic: 'Movimentos Culturais', discipline: 'História de Pernambuco', difficulty: 'Difícil', year: 2024,
        prompt: "O Movimento Armorial, com forte ligação com a cultura pernambucana, teve como um de seus principais idealizadores:",
        options: ["Clarice Lispector.", "João Cabral de Melo Neto.", "Ariano Suassuna.", "Manuel Bandeira."],
        answer: "Ariano Suassuna.",
    },
    {
        id: 14, topic: 'Tabela Verdade', discipline: 'Raciocínio Lógico', difficulty: 'Média', year: 2024,
        prompt: "A proposição 'Se o sol é quente, então a terra é redonda' é falsa apenas quando:",
        options: ["O sol é quente e a terra é redonda.", "O sol não é quente ou a terra é redonda.", "O sol é quente e a terra não é redonda.", "O sol não é quente e a terra não é redonda."],
        answer: "O sol é quente e a terra não é redonda.",
    },
    {
        id: 15, topic: 'Negação de Proposição', discipline: 'Raciocínio Lógico', difficulty: 'Fácil', year: 2023,
        prompt: "A negação da proposição 'João estuda e Maria trabalha' é:",
        options: ["João não estuda ou Maria não trabalha.", "João não estuda e Maria não trabalha.", "Se João estuda, então Maria não trabalha.", "João estuda ou Maria trabalha."],
        answer: "João não estuda ou Maria não trabalha.",
    },
    {
        id: 16, topic: 'Contagem', discipline: 'Raciocínio Lógico', difficulty: 'Difícil', year: 2022,
        prompt: "De quantas maneiras diferentes 5 pessoas podem se sentar em 5 cadeiras enfileiradas?",
        options: ["10.", "25.", "120.", "60."],
        answer: "120.", // 5! = 120
    },
    {
        id: 17, topic: 'Sistemas Operacionais', discipline: 'Informática', difficulty: 'Média', year: 2024,
        prompt: "Qual atalho de teclado é comumente usado no Windows para bloquear a tela do computador?",
        options: ["Alt + F4.", "Ctrl + Esc.", "Tecla Windows + L.", "Ctrl + Alt + Del."],
        answer: "Tecla Windows + L.",
    },
    {
        id: 18, topic: 'Conceitos de Internet', discipline: 'Informática', difficulty: 'Fácil', year: 2023,
        prompt: "O protocolo responsável por resolver nomes de domínio (URL) para endereços IP é o:",
        options: ["HTTP.", "FTP.", "SMTP.", "DNS."],
        answer: "DNS.",
    },
    {
        id: 19, topic: 'Criptografia', discipline: 'Informática', difficulty: 'Difícil', year: 2022,
        prompt: "A criptografia assimétrica utiliza:",
        options: ["A mesma chave para cifrar e decifrar.", "Uma chave pública para cifrar e uma chave privada para decifrar.", "Apenas chaves privadas.", "Uma chave para a cifragem e duas chaves para a decifragem."],
        answer: "Uma chave pública para cifrar e uma chave privada para decifrar.",
    },
    {
        id: 20, topic: 'Princípios Fundamentais', discipline: 'Direitos e Garantias Fundamentais', difficulty: 'Média', year: 2023,
        prompt: "A República Federativa do Brasil, formada pela união indissolúvel dos Estados e Municípios e do Distrito Federal, constitui-se em Estado Democrático de Direito e tem como fundamentos, EXCETO:",
        options: ["A soberania.", "A cidadania.", "A dignidade da pessoa humana.", "A concessão de privilégios aos militares."],
        answer: "A concessão de privilégios aos militares.",
    },
    {
        id: 21, topic: 'Remédios Constitucionais', discipline: 'Direitos e Garantias Fundamentais', difficulty: 'Fácil', year: 2022,
        prompt: "Qual remédio constitucional é cabível para proteger direito líquido e certo não amparado por Habeas Corpus ou Habeas Data?",
        options: ["Ação Popular.", "Mandado de Injunção.", "Mandado de Segurança.", "Ação Civil Pública."],
        answer: "Mandado de Segurança.",
    },
    {
        id: 22, topic: 'Estatuto PMPE (Lei 6.784/74)', discipline: 'Direitos e Garantias Fundamentais', difficulty: 'Difícil', year: 2024,
        prompt: "Conforme o Estatuto dos Policiais Militares do Estado de Pernambuco, a praça com mais de 10 anos de serviço será agregada quando:",
        options: ["For presa em flagrante delito.", "Estiver concorrendo a cargo eletivo.", "For condenada a pena privativa de liberdade.", "Estiver respondendo a inquérito policial militar."],
        answer: "Estiver concorrendo a cargo eletivo.",
    },
    {
        id: 23, topic: 'Ortografia', discipline: 'Língua Portuguesa', difficulty: 'Fácil', year: 2024,
        prompt: "Assinale a alternativa em que todas as palavras estão grafadas corretamente.",
        options: ["Exceção, ascenção, pretenção.", "Excessão, ascensão, pretensão.", "Exceção, ascensão, pretensão.", "Ecessão, assenção, pretenção."],
        answer: "Exceção, ascensão, pretensão.",
    },
    {
        id: 24, topic: 'Garantias Fundamentais', discipline: 'Direitos e Garantias Fundamentais', difficulty: 'Média', year: 2023,
        prompt: "Ninguém será levado à prisão ou nela mantido, quando a lei admitir a liberdade provisória, com ou sem fiança. Este princípio está contido no:",
        options: ["Art. 5º, LXVI da CF/88.", "Art. 144, § 5º da CF/88.", "Art. 37, 'caput' da CF/88.", "Art. 93, IX da CF/88."],
        answer: "Art. 5º, LXVI da CF/88.",
    },
    {
        id: 25, topic: 'Sistemas Operacionais', discipline: 'Informática', difficulty: 'Difícil', year: 2024,
        prompt: "No Linux, qual comando é usado para alterar as permissões de acesso a um arquivo?",
        options: ["ls.", "mkdir.", "chmod.", "chown."],
        answer: "chmod.",
    }
];

// --- COMPONENTES UI BÁSICOS ---

const Button = ({ children, primary = false, onClick, className = '' }) => (
    <button
        onClick={onClick}
        className={`px-6 py-3 font-semibold rounded-xl transition-all duration-200 shadow-md focus:outline-none focus:ring-4 ${primary
            ? 'bg-blue-600 text-white hover:bg-blue-700 focus:ring-blue-500/50'
            : 'bg-white text-gray-800 border border-gray-300 hover:bg-gray-100 focus:ring-gray-300/50'
            } ${className}`}
    >
        {children}
    </button>
);

const Card = ({ title, children, className = '' }) => (
    <div className={`bg-white p-6 rounded-2xl shadow-xl border border-gray-100 ${className}`}>
        {title && <h2 className="text-2xl font-bold text-gray-800 mb-4 border-b pb-2">{title}</h2>}
        {children}
    </div>
);

const IconButton = ({ icon, onClick, className = '' }) => (
    <button onClick={onClick} className={`p-2 rounded-full hover:bg-blue-100 text-blue-600 transition-colors ${className}`}>
        {icon}
    </button>
);

// --- COMPONENTE DE NAVEGAÇÃO ---

const Navbar = ({ activeView, setActiveView, isAuthReady }) => {
    const navItems = [
        { key: 'home', label: 'Início', icon: '🏠' },
        { key: 'questions', label: 'Questões', icon: '📚' },
        { key: 'simulado', label: 'Simulado', icon: '📝' },
        { key: 'profile', label: 'Perfil', icon: '👤' },
        { key: 'forum', label: 'Fórum', icon: '💬' },
    ];

    return (
        <nav className="fixed top-0 left-0 right-0 bg-white shadow-lg z-50">
            <div className="container mx-auto px-4 py-3 flex justify-between items-center">
                <div className="text-3xl font-extrabold text-blue-800 flex items-center">
                    <span className="mr-2">🛡️</span> Aprova PMPE
                </div>
                <div className="hidden md:flex space-x-2">
                    {navItems.map(item => (
                        <Button
                            key={item.key}
                            primary={activeView === item.key}
                            onClick={() => setActiveView(item.key)}
                            className="text-sm py-2 px-4"
                        >
                            {item.icon} {item.label}
                        </Button>
                    ))}
                </div>
                <div className="md:hidden">
                    <select
                        value={activeView}
                        onChange={(e) => setActiveView(e.target.value)}
                        className="p-2 border border-gray-300 rounded-lg focus:ring-blue-500 focus:border-blue-500"
                    >
                        {navItems.map(item => (
                            <option key={item.key} value={item.key}>
                                {item.icon} {item.label}
                            </option>
                        ))}
                    </select>
                </div>
                {isAuthReady && <span className="text-sm text-gray-500">Logado: {initialAuthToken ? 'Sim' : 'Anônimo'}</span>}
            </div>
        </nav>
    );
};

// --- COMPONENTE DE AUTENTICAÇÃO E FIRESTORE ---

const useFirebase = () => {
    const [db, setDb] = useState(null);
    const [auth, setAuth] = useState(null);
    const [userId, setUserId] = useState(null);
    const [isAuthReady, setIsAuthReady] = useState(false);

    useEffect(() => {
        if (!firebaseConfig.apiKey) {
            console.warn("Firebase config not available. Running in mock mode.");
            setIsAuthReady(true);
            setUserId(crypto.randomUUID());
            return;
        }

        try {
            const app = initializeApp(firebaseConfig);
            const firestore = getFirestore(app);
            const authService = getAuth(app);

            setDb(firestore);
            setAuth(authService);

            // 1. Authentication
            const unsubscribeAuth = onAuthStateChanged(authService, (user) => {
                const currentUserId = user?.uid || crypto.randomUUID();
                setUserId(currentUserId);
                setIsAuthReady(true);
                console.log(`Auth state changed. User ID: ${currentUserId}`);
            });

            // Sign in with custom token or anonymously
            const signIn = async () => {
                try {
                    if (initialAuthToken) {
                        await signInWithCustomToken(authService, initialAuthToken);
                        console.log("Signed in with custom token.");
                    } else {
                        await signInAnonymously(authService);
                        console.log("Signed in anonymously.");
                    }
                } catch (error) {
                    console.error("Firebase Sign-In failed:", error);
                    // Fallback to anonymous if custom token fails
                    await signInAnonymously(authService);
                }
            };

            signIn();
            return () => unsubscribeAuth();

        } catch (error) {
            console.error("Firebase Initialization Failed:", error);
            setIsAuthReady(true); // Still set ready for mock userId fallback
            setUserId(crypto.randomUUID());
        }
    }, []);

    const saveSimuladoResult = async (resultData) => {
        if (!db || !userId) {
            console.warn("Firestore not ready. Result not saved.");
            return;
        }
        try {
            const path = `artifacts/${appId}/users/${userId}/simuladoHistory`;
            await addDoc(collection(db, path), {
                ...resultData,
                timestamp: serverTimestamp()
            });
            console.log("Simulado result saved successfully.");
        } catch (error) {
            console.error("Error saving simulado result to Firestore:", error);
        }
    };

    return { db, userId, isAuthReady, saveSimuladoResult };
};

// --- VIEWS ---

// 1. HOME PAGE
const HomePage = ({ setActiveView }) => (
    <div className="text-center py-20 bg-gray-50 rounded-xl shadow-inner">
        <h1 className="text-5xl font-extrabold text-blue-800 mb-4">
            Aprova PMPE <span className="text-blue-500">com IA</span>
        </h1>
        <p className="text-xl text-gray-600 mb-8">Treine com inteligência artificial e conquiste sua farda em Pernambuco.</p>
        <div className="flex justify-center space-x-4">
            <Button primary onClick={() => setActiveView('simulado')}>
                📝 Fazer Simulado
            </Button>
            <Button onClick={() => setActiveView('questions')}>
                📚 Explorar Questões
            </Button>
        </div>
        <div className="mt-12 max-w-3xl mx-auto">
            <Card title="Como Funciona a IA de Curadoria?" className="text-left">
                <p className="text-gray-700">
                    Nossa plataforma utiliza a IA (Gemini) para dois propósitos cruciais:
                </p>
                <ul className="list-disc list-inside mt-4 space-y-2 text-gray-600">
                    <li>**Curadoria de Conteúdo:** Filtra e classifica automaticamente questões de fontes confiáveis, alinhando-as 100% com o edital de Nível Médio da PMPE.</li>
                    <li>**Explicações Inteligentes:** Após resolver uma questão, a IA gera uma explicação completa e didática, citando as referências legais (CF/88, Leis Estaduais) para garantir a fundamentação.</li>
                </ul>
                <p className="text-blue-600 font-semibold mt-4">
                    Estude de forma focada e fundamentada.
                </p>
            </Card>
        </div>
    </div>
);

// 2. BANK OF QUESTIONS (Explorar Questões)
const QuestionsBank = () => {
    const [selectedDiscipline, setSelectedDiscipline] = useState('Todos');
    const [explanation, setExplanation] = useState('');
    const [loading, setLoading] = useState(false);
    const [selectedQuestion, setSelectedQuestion] = useState(null);

    const filteredQuestions = MOCK_QUESTIONS.filter(q =>
        selectedDiscipline === 'Todos' || q.discipline === selectedDiscipline
    );

    const handleExplain = async (question) => {
        if (loading) return;
        setSelectedQuestion(question);
        setExplanation('');
        setLoading(true);

        const prompt = `Gere uma explicação completa e didática para a resposta correta desta questão do concurso PMPE, citando a regra ou lei pertinente (se aplicável): \n\nQuestão: "${question.prompt}"\nResposta Correta: "${question.answer}"\nDisciplina: ${question.discipline}`;
        const systemPrompt = "Você é um professor altamente qualificado e especialista em concursos da Polícia Militar de Pernambuco (PMPE). Seu objetivo é fornecer uma explicação didática, precisa e completa para a resposta de uma questão, citando as fontes legais ou doutrinárias relevantes para o concurso PMPE (nível médio). Responda em Português.";

        try {
            const { text, sources } = await callGeminiApi(prompt, systemPrompt, true);
            let finalExplanation = text;

            if (sources.length > 0) {
                finalExplanation += "\n\n---\n\n**Fontes de Pesquisa (IA):**\n";
                sources.forEach((s, index) => {
                    finalExplanation += `${index + 1}. [${s.title || 'Referência Web'}](${s.uri})\n`;
                });
            }

            setExplanation(finalExplanation);
        } catch (error) {
            setExplanation("Erro ao gerar explicação. Tente novamente mais tarde.");
        } finally {
            setLoading(false);
        }
    };

    return (
        <Card title="📚 Banco de Questões Inteligente" className="min-h-[500px]">
            <div className="flex flex-col md:flex-row md:space-x-4 mb-6">
                <label className="block text-sm font-medium text-gray-700 mb-2">Filtrar por Disciplina:</label>
                <select
                    value={selectedDiscipline}
                    onChange={(e) => {
                        setSelectedDiscipline(e.target.value);
                        setExplanation('');
                        setSelectedQuestion(null);
                    }}
                    className="p-2 border border-gray-300 rounded-lg focus:ring-blue-500 focus:border-blue-500"
                >
                    <option value="Todos">Todas as Disciplinas</option>
                    {DISCIPLINES.map(d => <option key={d} value={d}>{d}</option>)}
                </select>
            </div>

            <div className="grid grid-cols-1 md:grid-cols-2 gap-6">
                <div>
                    <h3 className="text-xl font-semibold mb-3 text-blue-600">Questões ({filteredQuestions.length})</h3>
                    <div className="space-y-4 max-h-[400px] overflow-y-auto pr-2">
                        {filteredQuestions.map((q) => (
                            <div key={q.id} className={`p-4 border rounded-xl cursor-pointer transition-shadow ${selectedQuestion?.id === q.id ? 'bg-blue-50 border-blue-400 shadow-lg' : 'hover:shadow-md bg-white'}`}
                                onClick={() => handleExplain(q)}
                            >
                                <p className="font-medium text-gray-800 line-clamp-2">{q.prompt}</p>
                                <span className="text-xs text-gray-500 mt-1 block">
                                    {q.discipline} | {q.difficulty} | {q.year}
                                </span>
                            </div>
                        ))}
                    </div>
                </div>

                <div className="border-l pl-6">
                    <h3 className="text-xl font-semibold mb-3 text-blue-600">Explicação da IA</h3>
                    {selectedQuestion && (
                        <div className="mb-4 p-4 border-b border-gray-200">
                            <p className="font-bold text-gray-800">Questão Selecionada:</p>
                            <p className="italic text-sm text-gray-600">"{selectedQuestion.prompt}"</p>
                            <p className="text-sm font-semibold text-green-600 mt-2">Resposta Correta: {selectedQuestion.answer}</p>
                        </div>
                    )}
                    <div className="min-h-[200px] bg-gray-50 p-4 rounded-xl text-gray-700 whitespace-pre-line">
                        {loading ? (
                            <div className="flex justify-center items-center h-full">
                                <span className="text-blue-500 animate-pulse">⚙️ A IA está gerando a explicação fundamentada...</span>
                            </div>
                        ) : explanation ? (
                            explanation
                        ) : (
                            <p className="text-center text-gray-400">Clique em uma questão para que a IA gere sua explicação.</p>
                        )}
                    </div>
                </div>
            </div>
        </Card>
    );
};

// 3. SIMULADO VIEWS
const SimuladoSetup = ({ startSimulado }) => {
    const [selectedDisciplines, setSelectedDisciplines] = useState(DISCIPLINES);
    const [count, setCount] = useState(10); // Aumentei o default para 10
    const [difficulty, setDifficulty] = useState('Média');

    const handleDisciplineToggle = (discipline) => {
        setSelectedDisciplines(prev =>
            prev.includes(discipline)
                ? prev.filter(d => d !== discipline)
                : [...prev, discipline]
        );
    };

    const handleStart = () => {
        if (selectedDisciplines.length === 0) {
            // Em vez de alert(), usamos um pop-up ou modal customizado.
            // Aqui, por simplicidade, vou manter uma mensagem na tela.
            console.log("Selecione pelo menos uma disciplina.");
            return;
        }
        startSimulado({ count: Number(count), disciplines: selectedDisciplines, difficulty });
    };

    return (
        <Card title="📝 Configurar Simulado Personalizado">
            <div className="space-y-6">
                <div>
                    <label className="block text-lg font-medium text-gray-700 mb-2">1. Disciplinas (Selecione o Foco)</label>
                    <div className="flex flex-wrap gap-2">
                        {DISCIPLINES.map(d => (
                            <button
                                key={d}
                                onClick={() => handleDisciplineToggle(d)}
                                className={`px-4 py-2 text-sm font-medium rounded-full transition-colors ${selectedDisciplines.includes(d)
                                    ? 'bg-blue-600 text-white shadow-md'
                                    : 'bg-gray-200 text-gray-700 hover:bg-gray-300'
                                    }`}
                            >
                                {d}
                            </button>
                        ))}
                    </div>
                </div>

                <div className="grid grid-cols-1 md:grid-cols-2 gap-6">
                    <div>
                        <label className="block text-lg font-medium text-gray-700 mb-2">2. Quantidade de Questões</label>
                        <input
                            type="number"
                            min="1"
                            max={MOCK_QUESTIONS.length}
                            value={count}
                            onChange={(e) => setCount(Math.min(MOCK_QUESTIONS.length, Math.max(1, Number(e.target.value))))}
                            className="w-full p-3 border border-gray-300 rounded-lg focus:ring-blue-500 focus:border-blue-500"
                        />
                        <small className="text-gray-500">Máximo disponível: {MOCK_QUESTIONS.length}</small>
                    </div>
                    <div>
                        <label className="block text-lg font-medium text-gray-700 mb-2">3. Nível de Dificuldade</label>
                        <select
                            value={difficulty}
                            onChange={(e) => setDifficulty(e.target.value)}
                            className="w-full p-3 border border-gray-300 rounded-lg focus:ring-blue-500 focus:border-blue-500"
                        >
                            <option value="Fácil">Fácil</option>
                            <option value="Média">Média</option>
                            <option value="Difícil">Difícil</option>
                        </select>
                    </div>
                </div>

                <Button primary onClick={handleStart} className="w-full text-lg mt-4">
                    🚀 Iniciar Simulado Agora!
                </Button>
            </div>
        </Card>
    );
};

const SimuladoQuiz = ({ questions, finishSimulado }) => {
    const [currentQuestionIndex, setCurrentQuestionIndex] = useState(0);
    const [answers, setAnswers] = useState(new Array(questions.length).fill(null));
    const [startTime] = useState(Date.now());
    const [showConfirm, setShowConfirm] = useState(false);

    const currentQuestion = questions[currentQuestionIndex];

    const handleAnswer = (option) => {
        const newAnswers = [...answers];
        newAnswers[currentQuestionIndex] = option;
        setAnswers(newAnswers);
    };

    const handleNext = () => {
        if (currentQuestionIndex < questions.length - 1) {
            setCurrentQuestionIndex(currentQuestionIndex + 1);
        }
    };

    const handlePrev = () => {
        if (currentQuestionIndex > 0) {
            setCurrentQuestionIndex(currentQuestionIndex - 1);
        }
    };

    const handleFinish = () => {
        const timeTaken = (Date.now() - startTime) / 1000; // Tempo em segundos
        const correctCount = questions.reduce((acc, q, index) => {
            return acc + (q.answer === answers[index] ? 1 : 0);
        }, 0);

        finishSimulado({
            correctCount,
            totalQuestions: questions.length,
            timeTaken,
            answeredQuestions: questions.map((q, index) => ({
                id: q.id,
                discipline: q.discipline,
                topic: q.topic,
                correct: q.answer === answers[index],
                userAnswer: answers[index],
                correctAnswer: q.answer
            }))
        });
    };

    const progress = (currentQuestionIndex + 1) / questions.length * 100;

    return (
        <Card title={`Questão ${currentQuestionIndex + 1} de ${questions.length}`} className="relative">
            {showConfirm && (
                <div className="absolute inset-0 bg-white/90 backdrop-blur-sm flex justify-center items-center z-10 rounded-2xl">
                    <Card title="Confirmar Finalização" className="w-96 text-center shadow-2xl">
                        <p className="mb-4">Deseja realmente finalizar o simulado?</p>
                        <p className="mb-6 text-sm text-red-500">Questões não respondidas serão contadas como erro.</p>
                        <div className="flex justify-center space-x-4">
                            <Button onClick={() => setShowConfirm(false)}>
                                Voltar ao Simulado
                            </Button>
                            <Button primary onClick={handleFinish}>
                                Finalizar e Ver Resultados
                            </Button>
                        </div>
                    </Card>
                </div>
            )}

            <div className="w-full bg-gray-200 rounded-full h-2.5 mb-4">
                <div className="bg-blue-600 h-2.5 rounded-full transition-all duration-500" style={{ width: `${progress}%` }}></div>
            </div>

            <p className="text-sm font-semibold text-blue-600 mb-4">{currentQuestion.discipline} - {currentQuestion.topic}</p>
            <p className="text-xl font-medium text-gray-800 mb-6">{currentQuestion.prompt}</p>

            <div className="space-y-3">
                {currentQuestion.options.map((option, index) => (
                    <button
                        key={index}
                        onClick={() => handleAnswer(option)}
                        className={`w-full text-left p-3 border rounded-xl transition-all duration-200 ${answers[currentQuestionIndex] === option
                            ? 'bg-blue-100 border-blue-600 ring-2 ring-blue-500 font-semibold'
                            : 'bg-gray-50 hover:bg-gray-100 border-gray-200'
                            }`}
                    >
                        {String.fromCharCode(65 + index)}. {option}
                    </button>
                ))}
            </div>

            <div className="flex justify-between mt-8 pt-4 border-t">
                <Button onClick={handlePrev} disabled={currentQuestionIndex === 0}>
                    ⬅️ Anterior
                </Button>
                {currentQuestionIndex < questions.length - 1 ? (
                    <Button primary onClick={handleNext}>
                        Próxima ➡️
                    </Button>
                ) : (
                    <Button primary onClick={() => setShowConfirm(true)} className="bg-green-600 hover:bg-green-700">
                        Finalizar Simulado
                    </Button>
                )}
            </div>
        </Card>
    );
};

const SimuladoResults = ({ results, resetSimulado, saveSimuladoResult, setActiveView, userId }) => {
    const { correctCount, totalQuestions, timeTaken, answeredQuestions } = results;
    const scorePercentage = (correctCount / totalQuestions) * 100;
    const timePerQuestion = (timeTaken / totalQuestions).toFixed(1);
    const [recommendations, setRecommendations] = useState('');
    const [loading, setLoading] = useState(false);
    const [saved, setSaved] = useState(false);

    const weakTopics = useMemo(() => {
        const wrongAnswers = answeredQuestions.filter(q => !q.correct);
        const topics = wrongAnswers.map(q => q.topic);
        // Contagem simples dos tópicos que o usuário errou
        const topicCounts = topics.reduce((acc, topic) => {
            acc[topic] = (acc[topic] || 0) + 1;
            return acc;
        }, {});
        return Object.keys(topicCounts).sort((a, b) => topicCounts[b] - topicCounts[a]).slice(0, 3);
    }, [answeredQuestions]);

    // Chama a IA para as recomendações
    useEffect(() => {
        const generateRecommendations = async () => {
            setLoading(true);
            const weakTopicsList = weakTopics.join(', ');
            const prompt = `Analise meu resultado (acertos: ${correctCount}/${totalQuestions}) e, com foco nos tópicos que preciso reforçar (${weakTopicsList}), gere 3 a 4 ações de estudo específicas e um breve resumo do conteúdo que devo revisar para o concurso PMPE. Seja um mentor de estudos, motivacional e focado na PMPE.`;
            const systemPrompt = "Você é um mentor de estudos de alta performance para o concurso da PMPE. Seu foco é analisar os pontos fracos do candidato e fornecer um plano de ação imediato, motivacional e ultra-focado no edital de nível médio. Use fontes reais para fundamentar a revisão.";

            try {
                const { text } = await callGeminiApi(prompt, systemPrompt, true);
                setRecommendations(text);
            } catch (error) {
                setRecommendations("Desculpe, a IA não conseguiu gerar recomendações. Reveja seus tópicos fracos manualmente.");
            } finally {
                setLoading(false);
            }
        };

        generateRecommendations();
    }, [correctCount, totalQuestions, weakTopics]);

    // Salva o resultado no Firestore
    const handleSaveResult = () => {
        if (!saved) {
            saveSimuladoResult({
                score: correctCount,
                totalQuestions: totalQuestions,
                timeTaken: timeTaken,
                disciplines: [...new Set(answeredQuestions.map(q => q.discipline))],
                weakTopics: weakTopics
            });
            setSaved(true);
        }
    };

    useEffect(() => {
        handleSaveResult(); // Salva automaticamente ao carregar
    }, []);

    return (
        <Card title="✅ Resultado do Simulado e Análise da IA">
            <div className="grid grid-cols-1 md:grid-cols-3 gap-6 text-center mb-8">
                <StatBox title="Acertos" value={`${correctCount}/${totalQuestions}`} color="text-green-600" />
                <StatBox title="Percentual" value={`${scorePercentage.toFixed(1)}%`} color="text-blue-600" />
                <StatBox title="Tempo Médio p/ Questão" value={`${timePerQuestion}s`} color="text-red-600" />
            </div>

            <Card title="🧠 Recomendações de Estudo da IA" className="bg-blue-50 border-blue-200">
                {loading ? (
                    <div className="flex flex-col items-center justify-center h-40">
                        <div className="animate-spin rounded-full h-12 w-12 border-b-2 border-blue-500"></div>
                        <p className="mt-4 text-blue-600 font-medium">Analisando seu desempenho e gerando plano de ação...</p>
                    </div>
                ) : (
                    <div className="whitespace-pre-line text-gray-700">{recommendations}</div>
                )}
            </Card>

            <div className="mt-8 pt-4 border-t flex justify-between">
                <Button primary onClick={() => setActiveView('profile')}>
                    👤 Ver Perfil e Histórico
                </Button>
                <Button onClick={resetSimulado} className="bg-yellow-500 text-white hover:bg-yellow-600">
                    Novo Simulado
                </Button>
            </div>
        </Card>
    );
};

const StatBox = ({ title, value, color }) => (
    <div className="p-4 bg-white rounded-xl shadow-md border">
        <p className="text-lg font-medium text-gray-500">{title}</p>
        <p className={`text-3xl font-bold ${color}`}>{value}</p>
    </div>
);

// 4. PERFIL DO ESTUDANTE
const Profile = ({ userId, db, isAuthReady, setActiveView }) => {
    const [history, setHistory] = useState([]);
    const [loading, setLoading] = useState(true);

    // Fetch Simulado History from Firestore
    useEffect(() => {
        if (!isAuthReady || !db || !userId) return;

        const path = `artifacts/${appId}/users/${userId}/simuladoHistory`;
        const q = query(collection(db, path), orderBy('timestamp', 'desc'));

        const unsubscribe = onSnapshot(q, (snapshot) => {
            const results = snapshot.docs.map(doc => ({
                id: doc.id,
                ...doc.data(),
                timestamp: doc.data().timestamp?.toDate() || new Date()
            }));
            setHistory(results);
            setLoading(false);
        }, (error) => {
            console.error("Error fetching simulado history:", error);
            setLoading(false);
        });

        return () => unsubscribe();
    }, [db, userId, isAuthReady]);

    const totalSimulados = history.length;
    const totalQuestions = history.reduce((sum, h) => sum + h.totalQuestions, 0);
    const totalCorrect = history.reduce((sum, h) => sum + h.score, 0);
    const avgScore = totalQuestions > 0 ? ((totalCorrect / totalQuestions) * 100).toFixed(1) : 0;
    const avgTime = totalQuestions > 0 ? (history.reduce((sum, h) => sum + h.timeTaken, 0) / totalQuestions).toFixed(1) : 0;

    return (
        <Card title="👤 Meu Painel de Performance" className="min-h-[500px]">
            <div className="mb-6">
                <p className="text-md font-medium text-gray-700">Seu ID de Usuário: <span className="font-mono text-sm bg-gray-100 p-1 rounded">{userId || 'Carregando...'}</span></p>
            </div>

            <div className="grid grid-cols-2 md:grid-cols-4 gap-4 mb-8">
                <StatBox title="Simulados Feitos" value={totalSimulados} color="text-blue-600" />
                <StatBox title="Questões Total" value={totalQuestions} color="text-gray-800" />
                <StatBox title="Acerto Médio" value={`${avgScore}%`} color="text-green-600" />
                <StatBox title="Tempo/Questão" value={`${avgTime}s`} color="text-red-600" />
            </div>

            <h3 className="text-xl font-bold text-gray-800 mb-4 border-b pb-2">Histórico Recente</h3>
            {loading ? (
                <div className="text-center py-10 text-blue-500">Carregando histórico...</div>
            ) : history.length === 0 ? (
                <div className="text-center py-10 text-gray-500">
                    <p>Você ainda não fez nenhum simulado. Comece agora!</p>
                    <Button primary onClick={() => setActiveView('simulado')} className="mt-4">Fazer Simulado</Button>
                </div>
            ) : (
                <div className="space-y-3 max-h-80 overflow-y-auto">
                    {history.map(h => (
                        <div key={h.id} className="p-4 bg-white border rounded-xl shadow-sm flex justify-between items-center hover:bg-gray-50 transition-colors">
                            <div>
                                <p className="font-semibold text-gray-800">
                                    Simulado de {h.disciplines.join(', ')}
                                </p>
                                <p className="text-sm text-gray-500">
                                    {h.timestamp.toLocaleDateString()} às {h.timestamp.toLocaleTimeString()}
                                </p>
                            </div>
                            <div className="text-right">
                                <p className={`text-xl font-bold ${h.score / h.totalQuestions > 0.7 ? 'text-green-600' : 'text-orange-500'}`}>
                                    {h.score}/{h.totalQuestions}
                                </p>
                                <p className="text-xs text-gray-500">
                                    {(h.score / h.totalQuestions * 100).toFixed(1)}%
                                </p>
                            </div>
                        </div>
                    ))}
                </div>
            )}
        </Card>
    );
};

// 5. FÓRUM DE DÚVIDAS (Simulado)
const Forum = () => {
    const [posts, setPosts] = useState([]);
    const [newPost, setNewPost] = useState('');
    const [loading, setLoading] = useState(false);
    const [isThinking, setIsThinking] = useState(false);

    // Mock data for the forum
    useEffect(() => {
        setPosts([
            { id: 1, user: 'Candidato_2025', text: 'Alguém sabe onde na CF/88 está a inafiançabilidade do crime de racismo? Tenho dificuldade em achar o artigo exato.', isIA: false, responses: [] },
            { id: 2, user: 'PMPE_Mentor_IA', text: 'Olá! A inafiançabilidade do crime de racismo está prevista no Art. 5º, inciso XLII, da Constituição Federal. É importante memorizar o "4 I" (Inafiançável e Imprescritível) para crimes como Racismo e Terrorismo (entre outros). Foco na Lei 7.716/89 (Lei do Racismo)!', isIA: true, responses: [] },
        ]);
    }, []);

    const handlePost = async () => {
        if (newPost.trim() === '') return;

        const newPostItem = { id: Date.now(), user: 'Você', text: newPost, isIA: false, responses: [], isNew: true };
        setPosts(prev => [newPostItem, ...prev]);
        setNewPost('');
        setIsThinking(true);

        // IA Moderador Inteligente - Sugerindo resposta
        const prompt = `Aja como um moderador inteligente e mentor de estudos da PMPE. Analise a seguinte dúvida de um estudante e gere uma resposta concisa, citando a fonte principal (lei, artigo, etc.) e oferecendo um macete ou dica de estudo. A dúvida é: "${newPost}"`;
        const systemPrompt = "Você é um moderador de fórum e mentor de estudos para o concurso PMPE. Responda à dúvida do estudante com uma explicação direta, focada e 100% correta. Resposta em Português.";

        try {
            const { text } = await callGeminiApi(prompt, systemPrompt, true);
            const iaResponse = {
                id: Date.now() + 1,
                user: 'PMPE_Mentor_IA',
                text: text,
                isIA: true,
                responses: [],
                isNew: true
            };
            setPosts(prev => [iaResponse, ...prev.slice(1)]); // Replace the temporary user post with the definitive one and add IA response

            // Since we can't update a nested array easily, we'll just prepend the IA response to the list.
            setPosts(prev => [iaResponse, prev[0], ...prev.slice(1).map(p => ({ ...p, isNew: false }))]);
        } catch (error) {
            console.error("IA Forum Error:", error);
            setPosts(prev => [{ ...prev[0], text: `${prev[0].text} (Erro na IA: não foi possível responder)`, isNew: false }, ...prev.slice(1)]);
        } finally {
            setIsThinking(false);
        }
    };

    return (
        <Card title="💬 Fórum de Dúvidas PMPE - Moderado por IA" className="min-h-[500px]">
            <div className="mb-6">
                <h3 className="text-xl font-semibold mb-2 text-blue-600">Postar Dúvida</h3>
                <textarea
                    value={newPost}
                    onChange={(e) => setNewPost(e.target.value)}
                    placeholder="Ex: Qual a diferença entre regência nominal e verbal? Onde isso cai mais na PMPE?"
                    rows="3"
                    className="w-full p-3 border border-gray-300 rounded-lg focus:ring-blue-500 focus:border-blue-500"
                ></textarea>
                <Button primary onClick={handlePost} disabled={newPost.trim() === '' || isThinking} className="mt-2">
                    {isThinking ? '🧠 IA Pensando...' : 'Publicar Dúvida'}
                </Button>
            </div>

            <h3 className="text-xl font-semibold mb-4 text-gray-800 border-b pb-2">Dúvidas Recentes</h3>
            <div className="space-y-6 max-h-[400px] overflow-y-auto pr-2">
                {posts.map(post => (
                    <div key={post.id} className={`p-4 rounded-xl transition-shadow ${post.isIA ? 'bg-blue-50 border-blue-200 shadow-md' : 'bg-white border border-gray-200'}`}>
                        <div className={`font-semibold mb-2 flex items-center ${post.isIA ? 'text-blue-700' : 'text-gray-800'}`}>
                            {post.isIA ? '🛡️ Mentor IA' : '👤 ' + post.user}
                            {post.isNew && <span className="ml-2 text-xs bg-green-500 text-white px-2 py-0.5 rounded-full">Novo!</span>}
                        </div>
                        <p className="text-gray-700 whitespace-pre-line">{post.text}</p>
                    </div>
                ))}
            </div>
        </Card>
    );
};

// --- MAIN APP COMPONENT ---

const App = () => {
    const [activeView, setActiveView] = useState('home');
    const [simuladoState, setSimuladoState] = useState({ mode: 'setup', questions: [], results: null });

    const { db, userId, isAuthReady, saveSimuladoResult } = useFirebase();

    const startSimulado = ({ count, disciplines, difficulty }) => {
        // Filtra e seleciona questões baseadas nas configurações (simulado de curadoria IA)
        let availableQuestions = MOCK_QUESTIONS
            .filter(q => disciplines.includes(q.discipline))
            .filter(q => difficulty === 'Média' || q.difficulty === difficulty); // Simplificação do filtro

        // Embaralha e limita a contagem
        availableQuestions = availableQuestions.sort(() => 0.5 - Math.random()).slice(0, count);

        setSimuladoState({ mode: 'quiz', questions: availableQuestions, results: null });
    };

    const finishSimulado = (results) => {
        setSimuladoState(prev => ({ ...prev, mode: 'results', results }));
    };

    const resetSimulado = () => {
        setSimuladoState({ mode: 'setup', questions: [], results: null });
        setActiveView('simulado');
    };

    const renderContent = () => {
        switch (activeView) {
            case 'home':
                return <HomePage setActiveView={setActiveView} />;
            case 'questions':
                return <QuestionsBank />;
            case 'simulado':
                if (simuladoState.mode === 'setup') {
                    return <SimuladoSetup startSimulado={startSimulado} />;
                } else if (simuladoState.mode === 'quiz') {
                    return <SimuladoQuiz questions={simuladoState.questions} finishSimulado={finishSimulado} />;
                } else if (simuladoState.mode === 'results') {
                    return (
                        <SimuladoResults
                            results={simuladoState.results}
                            resetSimulado={resetSimulado}
                            saveSimuladoResult={saveSimuladoResult}
                            setActiveView={setActiveView}
                            userId={userId}
                        />
                    );
                }
                break;
            case 'profile':
                return <Profile userId={userId} db={db} isAuthReady={isAuthReady} setActiveView={setActiveView} />;
            case 'forum':
                return <Forum />;
            default:
                return <HomePage setActiveView={setActiveView} />;
        }
    };

    return (
        <div className="min-h-screen bg-gray-100 font-sans">
            <Navbar activeView={activeView} setActiveView={setActiveView} isAuthReady={isAuthReady} />
            <main className="container mx-auto px-4 pt-24 pb-8">
                {renderContent()}
            </main>
            <footer className="text-center py-6 text-sm text-gray-500 border-t mt-8">
                Aprova PMPE - Seu futuro na PMPE começa aqui. Tecnologia com Firebase & Gemini.
            </footer>
        </div>
    );
};

export default App;
