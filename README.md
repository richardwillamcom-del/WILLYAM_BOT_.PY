para fazer o bot fucionar .env

# ════════════════════════════════════════════════
#         WILLYAM BOT — Arquivo de Configuração
#         NUNCA compartilhe este arquivo!
# ════════════════════════════════════════════════

# Token do seu Bot do Discord
# Obtenha em: https://discord.com/developers/applications
DISCORD_TOKEN=colocar token aqui

# Chave da API da Anthropic (Claude)
# Obtenha em: https://console.anthropic.com/
ANTHROPIC_API_KEY=colocar key aqui

# Seu ID numérico do Discord (clique direito no seu perfil > Copiar ID)
# Necessário para comandos de admin como /adicionar-saldo
DONO_BOT_ID=colocar id aqui




requerimento

discord.py>=2.3.0
anthropic>=0.25.0
python-dotenv>=1.0.0


OBS: CRI O REQUERIMENTO, O BOT E A .ENV SEPARADAS 




"""
╔══════════════════════════════════════════════════════════════════╗
║              WILLYAM BOT — Bot de Estudos ENEM v2.0             ║
║          Refatorado com segurança, novos comandos e IA          ║
╚══════════════════════════════════════════════════════════════════╝

COMANDOS DISPONÍVEIS: esse codigo é antigo o novo é to modificando ainda 
─────────────────────────────────────────────────────────────────
🏗️  ADMINISTRAÇÃO:
    /organizar        → Monta toda a estrutura do servidor
    /cargos           → Cria os cargos do servidor
    /executar         → Controle total por linguagem natural (IA)
    /criar-categoria  → IA cria categoria com canais pelo tema
    /limpar           → Apaga mensagens do canal (máx. 100)
    /avisar           → Envia aviso oficial a um usuário
    /regras           → Posta as regras no canal
    /setup-boas-vindas → Configura mensagem de boas-vindas

🤖  IA E ESTUDOS:
    /ia               → Pergunta livre para a IA
    /assuntos         → Posta e fixa assuntos do ENEM no canal
    /resumo           → Gera resumo de um tópico com IA
    /flashcards       → Gera flashcards interativos de um tema
    /redação          → IA avalia ou gera estrutura de redação
    /simulado         → Gera questão estilo ENEM com IA
    /cronograma       → Gera cronograma de estudos personalizado
    /dica-enem        → Dica diária de estudo para o ENEM

💰  CRÉDITOS (Público):
    /saldo            → Vê saldo de créditos de IA do servidor

💰  CRÉDITOS (Dono do Bot):
    /adicionar-saldo  → Adiciona créditos a um servidor
    /remover-saldo    → Remove créditos de um servidor
    /consultar-servidor → Relatório completo de uso

📊  UTILIDADE:
    /info             → Informações do servidor
    /perfil           → Perfil de estudos do usuário
    /meta             → Define uma meta de estudos
    /ranking          → Ranking de participação do servidor
    /sorteio          → Sorteia um membro do servidor
    /calcular         → Calcula nota do ENEM com IA
─────────────────────────────────────────────────────────────────
"""
import os
import re
import json
import random
import sqlite3
from datetime import datetime, timedelta
import discord
from discord import app_commands
from discord.ext import commands, tasks
import anthropic

# ==============================================================================
# CARREGAMENTO COMPLETO DAS CONFIGURAÇÕES DO ARQUIVO .ENV
# ==============================================================================
from dotenv import load_dotenv
load_dotenv()

# ==============================================================================
# CONFIGURAÇÃO – Variáveis de Ambiente Seguras
# ==============================================================================
DISCORD_TOKEN     = os.getenv("DISCORD_TOKEN")
ANTHROPIC_API_KEY = os.getenv("ANTHROPIC_API_KEY")

try:
    # Lê o seu ID numérico do .env (Garante o funcionamento do /adicionar-saldo)
    DONO_BOT_ID   = int(os.getenv("DONO_BOT_ID", "0"))
except ValueError:
    DONO_BOT_ID   = 0

# Validação crítica — encerra se credenciais não estiverem configuradas
if not DISCORD_TOKEN:
    raise ValueError("❌ DISCORD_TOKEN não encontrado no ambiente. Configure o arquivo .env!")
if not ANTHROPIC_API_KEY:
    raise ValueError("❌ ANTHROPIC_API_KEY não encontrada no ambiente. Configure o arquivo .env!")
if DONO_BOT_ID == 0:
    print("⚠️  [AVISO] DONO_BOT_ID não configurado. Comandos financeiros de admin estarão desabilitados.")

# Constantes financeiras
PRECO_INPUT_USD  = 3.00   # por milhão de tokens
PRECO_OUTPUT_USD = 15.00  # por milhão de tokens
COTACAO_USD_BRL  = 5.70   # Atualize conforme necessário

# Palavras proibidas (adicione conforme necessário)
PALAVRAS_PROIBIDAS: list[str] = []

# Termos de injeção de prompt (anti-hack)
TERMOS_INJECAO = [
    "ignore as instruções anteriores",
    "ignore suas diretrizes",
    "ignore previous instructions",
    "atue como um novo sistema",
    "novo prompt",
    "esqueca tudo",
    "forget your instructions",
    "jailbreak",
    "dan mode",
]

# ══════════════════════════════════════════════════════════════════
# BANCO DE DADOS SQLite — Com gerenciadores de contexto (anti-lock)
# ══════════════════════════════════════════════════════════════════

def _timestamp() -> str:
    return datetime.now().strftime("%Y-%m-%d %H:%M:%S")

def init_db():
    """Inicializa todas as tabelas do banco de dados."""
    try:
        with sqlite3.connect("banco.db") as conn:
            c = conn.cursor()
            c.executescript("""
                CREATE TABLE IF NOT EXISTS servidores (
                    guild_id      TEXT PRIMARY KEY,
                    saldo_reais   REAL DEFAULT 0.0,
                    premium_ativo INTEGER DEFAULT 0
                );
                CREATE TABLE IF NOT EXISTS historico_consumo (
                    id                INTEGER PRIMARY KEY AUTOINCREMENT,
                    guild_id          TEXT,
                    comando           TEXT,
                    tokens_utilizados INTEGER,
                    custo_calculado   REAL,
                    data              TEXT
                );
                CREATE TABLE IF NOT EXISTS perfis_usuarios (
                    user_id    TEXT PRIMARY KEY,
                    guild_id   TEXT,
                    meta       TEXT,
                    xp         INTEGER DEFAULT 0,
                    mensagens  INTEGER DEFAULT 0,
                    ultimo_uso TEXT
                );
                CREATE TABLE IF NOT EXISTS avisos_usuarios (
                    id        INTEGER PRIMARY KEY AUTOINCREMENT,
                    guild_id  TEXT,
                    user_id   TEXT,
                    motivo    TEXT,
                    moderador TEXT,
                    data      TEXT
                );
            """)
            conn.commit()
        print(f"[{_timestamp()}] ✅ Banco de dados inicializado com sucesso.")
    except sqlite3.Error as e:
        print(f"[{_timestamp()}] ❌ ERRO DB (init): {e}")


def get_saldo(guild_id: str) -> float:
    try:
        with sqlite3.connect("banco.db") as conn:
            c = conn.cursor()
            c.execute("SELECT saldo_reais FROM servidores WHERE guild_id = ?", (guild_id,))
            row = c.fetchone()
            return row[0] if row else 0.0
    except sqlite3.Error as e:
        print(f"[{_timestamp()}] ❌ ERRO DB (get_saldo): {e}")
        return 0.0


def garantir_servidor(guild_id: str):
    try:
        with sqlite3.connect("banco.db") as conn:
            conn.execute("INSERT OR IGNORE INTO servidores (guild_id) VALUES (?)", (guild_id,))
            conn.commit()
    except sqlite3.Error as e:
        print(f"[{_timestamp()}] ❌ ERRO DB (garantir_servidor): {e}")


def atualizar_saldo(guild_id: str, delta: float):
    try:
        with sqlite3.connect("banco.db") as conn:
            conn.execute("""
                INSERT INTO servidores (guild_id, saldo_reais) VALUES (?, ?)
                ON CONFLICT(guild_id) DO UPDATE SET saldo_reais = saldo_reais + excluded.saldo_reais
            """, (guild_id, delta))
            conn.commit()
    except sqlite3.Error as e:
        print(f"[{_timestamp()}] ❌ ERRO DB (atualizar_saldo): {e}")


def registrar_consumo(guild_id: str, comando: str, tokens: int, custo: float):
    try:
        with sqlite3.connect("banco.db") as conn:
            conn.execute("""
                INSERT INTO historico_consumo (guild_id, comando, tokens_utilizados, custo_calculado, data)
                VALUES (?, ?, ?, ?, ?)
            """, (guild_id, comando, tokens, custo, _timestamp()))
            conn.commit()
    except sqlite3.Error as e:
        print(f"[{_timestamp()}] ❌ ERRO DB (registrar_consumo): {e}")


def get_total_tokens(guild_id: str) -> tuple:
    try:
        with sqlite3.connect("banco.db") as conn:
            c = conn.cursor()
            c.execute("""
                SELECT COALESCE(SUM(tokens_utilizados),0), COALESCE(SUM(custo_calculado),0)
                FROM historico_consumo WHERE guild_id = ?
            """, (guild_id,))
            row = c.fetchone()
            return row if row else (0, 0.0)
    except sqlite3.Error as e:
        print(f"[{_timestamp()}] ❌ ERRO DB (get_total_tokens): {e}")
        return (0, 0.0)


def get_perfil(user_id: str, guild_id: str) -> dict:
    try:
        with sqlite3.connect("banco.db") as conn:
            c = conn.cursor()
            c.execute("SELECT meta, xp, mensagens FROM perfis_usuarios WHERE user_id = ? AND guild_id = ?",
                      (user_id, guild_id))
            row = c.fetchone()
            if row:
                return {"meta": row[0], "xp": row[1], "mensagens": row[2]}
            return {"meta": "Não definida", "xp": 0, "mensagens": 0}
    except sqlite3.Error as e:
        print(f"[{_timestamp()}] ❌ ERRO DB (get_perfil): {e}")
        return {"meta": "Erro", "xp": 0, "mensagens": 0}


def atualizar_xp(user_id: str, guild_id: str, delta_xp: int = 1, delta_msg: int = 1):
    try:
        with sqlite3.connect("banco.db") as conn:
            conn.execute("""
                INSERT INTO perfis_usuarios (user_id, guild_id, xp, mensagens, ultimo_uso)
                VALUES (?, ?, ?, ?, ?)
                ON CONFLICT(user_id) DO UPDATE SET
                    xp = xp + ?,
                    mensagens = mensagens + ?,
                    ultimo_uso = ?
            """, (user_id, guild_id, delta_xp, delta_msg, _timestamp(),
                  delta_xp, delta_msg, _timestamp()))
            conn.commit()
    except sqlite3.Error as e:
        print(f"[{_timestamp()}] ❌ ERRO DB (atualizar_xp): {e}")


def definir_meta(user_id: str, guild_id: str, meta: str):
    try:
        with sqlite3.connect("banco.db") as conn:
            conn.execute("""
                INSERT INTO perfis_usuarios (user_id, guild_id, meta)
                VALUES (?, ?, ?)
                ON CONFLICT(user_id) DO UPDATE SET meta = ?
            """, (user_id, guild_id, meta, meta))
            conn.commit()
    except sqlite3.Error as e:
        print(f"[{_timestamp()}] ❌ ERRO DB (definir_meta): {e}")


def registrar_aviso(guild_id: str, user_id: str, motivo: str, moderador: str):
    try:
        with sqlite3.connect("banco.db") as conn:
            conn.execute("""
                INSERT INTO avisos_usuarios (guild_id, user_id, motivo, moderador, data)
                VALUES (?, ?, ?, ?, ?)
            """, (guild_id, user_id, motivo, moderador, _timestamp()))
            conn.commit()
    except sqlite3.Error as e:
        print(f"[{_timestamp()}] ❌ ERRO DB (registrar_aviso): {e}")


def get_avisos(guild_id: str, user_id: str) -> list:
    try:
        with sqlite3.connect("banco.db") as conn:
            c = conn.cursor()
            c.execute("""
                SELECT motivo, moderador, data FROM avisos_usuarios
                WHERE guild_id = ? AND user_id = ? ORDER BY data DESC LIMIT 5
            """, (guild_id, user_id))
            return c.fetchall()
    except sqlite3.Error as e:
        print(f"[{_timestamp()}] ❌ ERRO DB (get_avisos): {e}")
        return []


def get_ranking(guild_id: str) -> list:
    try:
        with sqlite3.connect("banco.db") as conn:
            c = conn.cursor()
            c.execute("""
                SELECT user_id, xp, mensagens FROM perfis_usuarios
                WHERE guild_id = ? ORDER BY xp DESC LIMIT 10
            """, (guild_id,))
            return c.fetchall()
    except sqlite3.Error as e:
        print(f"[{_timestamp()}] ❌ ERRO DB (get_ranking): {e}")
        return []

# ══════════════════════════════════════════════════════════════════
# ESTRUTURA COMPLETA DO SERVIDOR
# ══════════════════════════════════════════════════════════════════

ESTRUTURA_SERVIDOR = [
    {
        "categoria": "📋 INFORMAÇÕES",
        "canais": [
            {"nome": "📌regras",          "tipo": "text",  "blindado": True},
            {"nome": "📢anúncios",        "tipo": "text",  "blindado": True},
            {"nome": "🙋apresentações",   "tipo": "text"},
            {"nome": "🤖bot-comandos",    "tipo": "text"},
        ]
    },
    {
        "categoria": "📚 ESTUDOS GERAIS",
        "canais": [
            {"nome": "💬geral-estudos",   "tipo": "text"},
            {"nome": "🗓cronogramas",     "tipo": "text"},
            {"nome": "🤖pergunte-a-ia",  "tipo": "text"},
            {"nome": "📎materiais",       "tipo": "text"},
            {"nome": "🎯metas-diarias",   "tipo": "text"},
        ]
    },
    {
        "categoria": "📖 LINGUAGENS E CÓDIGOS",
        "canais": [
            {"nome": "📖português",       "tipo": "text"},
            {"nome": "📚literatura",      "tipo": "text"},
            {"nome": "✍️redação",         "tipo": "text"},
            {"nome": "🇺🇸inglês",         "tipo": "text"},
            {"nome": "🇪🇸espanhol",       "tipo": "text"},
            {"nome": "🔊voz-português",   "tipo": "voice"},
            {"nome": "🔊voz-literatura",  "tipo": "voice"},
            {"nome": "🎙️debate-redação",  "tipo": "voice"},
        ]
    },
    {
        "categoria": "🔢 CIÊNCIAS EXATAS",
        "canais": [
            {"nome": "🔢matemática",      "tipo": "text"},
            {"nome": "⚗️química",         "tipo": "text"},
            {"nome": "🔬física",          "tipo": "text"},
            {"nome": "🔊voz-exatas-1",    "tipo": "voice"},
            {"nome": "🔊voz-exatas-2",    "tipo": "voice"},
        ]
    },
    {
        "categoria": "🌍 CIÊNCIAS HUMANAS",
        "canais": [
            {"nome": "📜história",        "tipo": "text"},
            {"nome": "🌍geografia",       "tipo": "text"},
            {"nome": "🏛️filosofia",       "tipo": "text"},
            {"nome": "👥sociologia",      "tipo": "text"},
            {"nome": "🔊voz-humanas-1",   "tipo": "voice"},
            {"nome": "🔊voz-humanas-2",   "tipo": "voice"},
        ]
    },
    {
        "categoria": "🧬 CIÊNCIAS DA NATUREZA",
        "canais": [
            {"nome": "🧬biologia",        "tipo": "text"},
            {"nome": "🔊voz-biologia",    "tipo": "voice"},
        ]
    },
    {
        "categoria": "💻 PROGRAMAÇÃO",
        "canais": [
            {"nome": "💻html",            "tipo": "text"},
            {"nome": "🎨css",             "tipo": "text"},
            {"nome": "📜javascript",      "tipo": "text"},
            {"nome": "🐍python",          "tipo": "text"},
            {"nome": "🔊dev-1",           "tipo": "voice"},
            {"nome": "🔊dev-2",           "tipo": "voice"},
        ]
    },
    {
        "categoria": "🎓 FACULDADE",
        "canais": [
            {"nome": "💬geral-faculdade", "tipo": "text"},
            {"nome": "📎materiais-facul", "tipo": "text"},
            {"nome": "🔊voz-faculdade",   "tipo": "voice"},
        ]
    },
    {
        "categoria": "🎸 MÚSICA E ARTES",
        "canais": [
            {"nome": "🎸teoria-violao",   "tipo": "text"},
            {"nome": "🎸teoria-guitarra", "tipo": "text"},
            {"nome": "🖌️técnicas-desenho","tipo": "text"},
            {"nome": "🖼️galeria",         "tipo": "text"},
            {"nome": "🔊sala-musica",     "tipo": "voice"},
        ]
    },
    {
        "categoria": "🏆 MOTIVAÇÃO",
        "canais": [
            {"nome": "🎯metas-da-semana", "tipo": "text"},
            {"nome": "✅conquistas",      "tipo": "text"},
            {"nome": "☕descanso",         "tipo": "text"},
            {"nome": "😂memes-estudantis","tipo": "text"},
        ]
    },
    {
        "categoria": "🔊 VOZ GERAL",
        "canais": [
            {"nome": "📖Sala de Estudos 1","tipo": "voice"},
            {"nome": "📖Sala de Estudos 2","tipo": "voice"},
            {"nome": "📖Sala de Estudos 3","tipo": "voice"},
            {"nome": "☕Bate-papo",         "tipo": "voice"},
        ]
    },
]

CARGOS = [
    {"nome": "👑 Dono",        "cor": discord.Color.gold(),                  "admin": True},
    {"nome": "🛡️ Moderador",   "cor": discord.Color.blue(),                  "admin": False},
    {"nome": "🧠 Monitor",     "cor": discord.Color.from_rgb(155, 89, 182),  "admin": False},
    {"nome": "🎓 Aluno",       "cor": discord.Color.green(),                 "admin": False},
    {"nome": "♂️ Homem",       "cor": discord.Color.from_rgb(52, 152, 219),  "admin": False},
    {"nome": "♀️ Mulher",      "cor": discord.Color.from_rgb(231, 76, 60),   "admin": False},
    {"nome": "📚 Estudante",   "cor": discord.Color.from_rgb(46, 204, 113),  "admin": False},
    {"nome": "🆕 Novato",      "cor": discord.Color.light_grey(),            "admin": False},
]

# ══════════════════════════════════════════════════════════════════
# ASSUNTOS ENEM COMPLETO (com flashcards e links por matéria)
# ══════════════════════════════════════════════════════════════════

ASSUNTOS = {
    "português": {
        "emoji": "📖",
        "topicos": [
            "**1. Interpretação de Texto** → Inferência, tema central, intenção do autor",
            "**2. Gêneros Textuais** → Crônica, conto, artigo, editorial, charge, tirinha",
            "**3. Gramática** → Concordância, regência, crase, pontuação, morfologia",
            "**4. Funções da Linguagem** → Referencial, emotiva, poética, fática, metalinguística",
            "**5. Figuras de Linguagem** → Metáfora, metonímia, ironia, hipérbole, eufemismo",
            "**6. Variação Linguística** → Dialetos, registros formal/informal, norma culta",
            "**7. Redação** → Dissertação argumentativa, repertório, proposta de intervenção",
            "**8. Literatura** → Modernismo, Romantismo, Realismo, Simbolismo, Barroco",
        ],
        "links": [
            ("▶️ Português — Descomplica", "https://www.youtube.com/@descomplica"),
            ("▶️ Professor Noslen — Gramática", "https://www.youtube.com/@ProfessorNoslen"),
            ("🌐 Brasil Escola — Português", "https://brasilescola.uol.com.br/portugues"),
            ("🌐 Toda Matéria — Língua Portuguesa", "https://www.todamateria.com.br/lingua-portuguesa/"),
        ],
        "flashcards": [
            ("O que é coesão textual?", "É a ligação entre os elementos do texto, garantida por conectivos, pronomes e outros mecanismos que dão continuidade à leitura."),
            ("O que é coerência textual?", "É a lógica interna do texto — as ideias devem ser compatíveis entre si e com a realidade para que o texto faça sentido."),
            ("Quais são as funções da linguagem?", "Referencial (informar), Emotiva (expressar sentimentos), Poética (estética), Fática (contato), Metalinguística (falar sobre a língua) e Conativa (convencer)."),
            ("O que é crase?", "A crase é a fusão da preposição 'a' com o artigo feminino 'a(s)' ou com os pronomes demonstrativos 'aquele(a)', 'aquilo'. Indicada pelo acento grave (`à`)."),
        ],
    },
    "literatura": {
        "emoji": "📚",
        "topicos": [
            "**1. Quinhentismo** → Literatura de informação, Carta de Pero Vaz de Caminha",
            "**2. Barroco** → Dualismo, Gregório de Matos, Padre Antônio Vieira",
            "**3. Arcadismo** → Neoclassicismo, Tomás Antônio Gonzaga, bucolismo",
            "**4. Romantismo** → Indianismo, Ultrarromantismo, José de Alencar, Álvares de Azevedo",
            "**5. Realismo e Naturalismo** → Machado de Assis, Eça de Queirós, crítica social",
            "**6. Parnasianismo e Simbolismo** → Olavo Bilac, Cruz e Sousa, subjetivismo",
            "**7. Pré-Modernismo** → Euclides da Cunha, Lima Barreto, Monteiro Lobato",
            "**8. Modernismo** → Semana de 22, Drummond, Guimarães Rosa, Clarice Lispector",
            "**9. Literatura Contemporânea** → Tendências atuais, poesia marginal",
        ],
        "links": [
            ("▶️ Professor Noslen — Literatura", "https://www.youtube.com/@ProfessorNoslen"),
            ("▶️ Me Salva! — Escolas Literárias", "https://www.youtube.com/@MeSalva"),
            ("🌐 Brasil Escola — Literatura", "https://brasilescola.uol.com.br/literatura"),
            ("🌐 Toda Matéria — Literatura Brasileira", "https://www.todamateria.com.br/literatura-brasileira/"),
        ],
        "flashcards": [
            ("O que foi o Romantismo?", "Uma escola literária do século XIX marcada pelo sentimentalismo, egocentrismo, fuga da realidade e idealização da natureza e da sociedade."),
            ("Quem foi Machado de Assis?", "Maior escritor brasileiro, fundador da Academia Brasileira de Letras, representante do Realismo/Naturalismo. Obras: 'Dom Casmurro', 'Memórias Póstumas de Brás Cubas'."),
            ("O que é o Modernismo?", "Movimento que buscou romper com o passado literário clássico, valorizando a linguagem coloquial, o cotidiano e a identidade nacional. Iniciou-se com a Semana de Arte Moderna de 1922."),
            ("O que caracteriza o Barroco?", "Conflito entre o sagrado e o profano, uso de antíteses e paradoxos, linguagem rebuscada. Principal autor: Gregório de Matos (o 'Boca do Inferno')."),
        ],
    },
    "filosofia": {
        "emoji": "🏛️",
        "topicos": [
            "**1. Mitologia vs. Filosofia** → Passagem do mito ao logos, pré-socráticos",
            "**2. Sócrates e Platão** → Maiêutica, mundo das ideias, alegoria da caverna",
            "**3. Aristóteles** → Lógica, ética, política, o justo meio",
            "**4. Helenismo** → Estoicismo, Epicurismo, Ceticismo, Cinismo",
            "**5. Filosofia Medieval** → Santo Agostinho (patrística), São Tomás (escolástica)",
            "**6. Filosofia Moderna** → Descartes (racionalismo), Locke e Hume (empirismo)",
            "**7. Iluminismo e Contratualismo** → Rousseau, Hobbes, Locke, Montesquieu",
            "**8. Filosofia Contemporânea** → Marx, Nietzsche, Existencialismo (Sartre)",
            "**9. Ética e Justiça** → Moral, bioética, direitos humanos, justiça social",
        ],
        "links": [
            ("▶️ Filosofia Animada — YouTube", "https://www.youtube.com/@FilosofiaAnimada"),
            ("▶️ Canal do Cortella — Filosofia", "https://www.youtube.com/@MarioCortella"),
            ("🌐 Brasil Escola — Filosofia", "https://brasilescola.uol.com.br/filosofia"),
            ("🌐 Toda Matéria — Filósofos", "https://www.todamateria.com.br/filosofia/"),
        ],
        "flashcards": [
            ("O que é a Alegoria da Caverna de Platão?", "Uma metáfora que descreve prisioneiros acorrentados numa caverna, enxergando apenas sombras. Representa a ilusão do mundo sensível vs. a verdade do mundo das ideias, acessível pela razão."),
            ("O que é o 'imperativo categórico' de Kant?", "O princípio moral supremo: age somente segundo a máxima pela qual possas querer que ela se torne uma lei universal. É o fundamento da ética kantiana."),
            ("Qual a diferença entre ética e moral?", "Moral é o conjunto de normas e costumes de uma sociedade. Ética é a reflexão filosófica crítica sobre esses costumes e valores — é a 'ciência da moral'."),
            ("O que é o Contrato Social de Rousseau?", "A teoria de que os indivíduos abrem mão de parte de sua liberdade natural em favor do Estado para garantir a convivência em sociedade e a vontade geral."),
        ],
    },
    "sociologia": {
        "emoji": "👥",
        "topicos": [
            "**1. Surgimento da Sociologia** → Comte, positivismo, método científico social",
            "**2. Karl Marx** → Materialismo histórico, luta de classes, mais-valia, alienação",
            "**3. Émile Durkheim** → Fato social, solidariedade, anomia, divisão do trabalho",
            "**4. Max Weber** → Ação social, tipos ideais, burocracia, ética protestante",
            "**5. Cultura e Indústria Cultural** → Escola de Frankfurt, Adorno, mass media",
            "**6. Ideologia e Poder** → Gramsci, hegemonia, aparelhos ideológicos do Estado",
            "**7. Desigualdade Social** → Classes sociais, mobilidade, racismo estrutural, gênero",
            "**8. Movimentos Sociais** → MST, feminismo, LGBTQIA+, movimento negro",
            "**9. Trabalho e Cidadania** → Direitos trabalhistas, uberização, democracia, Estado",
        ],
        "links": [
            ("▶️ Sociologia com o Prof. Leone — YouTube", "https://www.youtube.com/@professorleone"),
            ("▶️ Descomplica — Sociologia ENEM", "https://www.youtube.com/@descomplica"),
            ("🌐 Brasil Escola — Sociologia", "https://brasilescola.uol.com.br/sociologia"),
            ("🌐 Toda Matéria — Sociologia", "https://www.todamateria.com.br/sociologia/"),
        ],
        "flashcards": [
            ("O que é 'fato social' para Durkheim?", "São maneiras de agir, pensar e sentir exteriores ao indivíduo, que exercem sobre ele um poder de coerção. São gerais, coletivos e independentes das vontades individuais."),
            ("O que é mais-valia para Marx?", "É a diferença entre o valor produzido pelo trabalhador e o salário que ele recebe. O capitalista se apropria dessa diferença, gerando a exploração do proletariado."),
            ("O que é ação social para Weber?", "É uma ação orientada pelo sentido que o indivíduo atribui ao comportamento dos outros. Weber classifica em: racional por fins, racional por valores, afetiva e tradicional."),
            ("O que é hegemonia para Gramsci?", "É a dominação cultural e ideológica de uma classe sobre outra, exercida não apenas pela força (coerção), mas pelo consenso — convencendo a sociedade de que seus valores são universais."),
        ],
    },
    "redação": {
        "emoji": "✍️",
        "topicos": [
            "**1. Estrutura da Dissertação** → Introdução, 2 parágrafos de desenvolvimento, conclusão",
            "**2. Competência 1 — Norma Culta** → Gramática, ortografia, concordância, pontuação",
            "**3. Competência 2 — Compreensão do Tema** → Não fugir ao tema, ler bem a proposta",
            "**4. Competência 3 — Argumentação** → Tese, argumentos coesos, lógica",
            "**5. Competência 4 — Coesão Textual** → Conectivos, referências, progressão textual",
            "**6. Competência 5 — Proposta de Intervenção** → Agente | Ação | Modo | Efeito | Detalhamento",
            "**7. Repertório Legitimado** → Dados, leis, filósofos, obras literárias, fatos históricos",
            "**8. Tese e Argumentação** → Argumentos por autoridade, exemplificação, contra-argumento",
            "**9. Introdução Impactante** → Citação, dado, questionamento, situação-problema",
            "**10. Redações Nota 1000** → Análise de modelos, erros comuns, revisão",
        ],
        "links": [
            ("▶️ Se Liga ENEM — Redação", "https://www.youtube.com/@seligaenem"),
            ("▶️ Redação com a Prof. Pamba", "https://www.youtube.com/@profapamba"),
            ("🌐 ENEM — Competências Oficiais (MEC)", "https://www.gov.br/inep/pt-br/areas-de-atuacao/avaliacao-e-exames-educacionais/enem"),
            ("🌐 Correção de Redações — Brasil Escola", "https://brasilescola.uol.com.br/redacao"),
        ],
        "flashcards": [
            ("Quais são as 5 competências da redação ENEM?", "1) Norma culta; 2) Compreensão do tema; 3) Seleção e organização de argumentos; 4) Coesão e coerência textual; 5) Proposta de intervenção social, detalhada e respeitando os direitos humanos."),
            ("O que deve ter a proposta de intervenção?", "Os 5 elementos obrigatórios: Agente (quem executa), Ação (o que fazer), Modo/Meio (como), Efeito (resultado esperado) e Detalhamento (especificação de algum dos anteriores)."),
            ("O que é repertório legitimado?", "São dados estatísticos, leis, citações filosóficas/literárias, fatos históricos e pesquisas científicas que sustentam os argumentos com credibilidade."),
            ("Como fazer uma boa introdução?", "Apresentar o tema com um repertório (citação, dado ou fato), contextualizar o problema e finalizar com uma tese clara que será defendida ao longo do texto."),
        ],
    },
    "matemática": {
        "emoji": "🔢",
        "topicos": [
            "**1. Números e Operações** → Conjuntos numéricos, potências, raízes, logaritmos",
            "**2. Álgebra** → Equações do 1º e 2º grau, sistemas, inequações, polinômios",
            "**3. Funções** → Linear, quadrática, exponencial, logarítmica, modular",
            "**4. Trigonometria** → Seno, cosseno, tangente, lei dos senos/cossenos",
            "**5. Geometria Plana** → Áreas, perímetros, triângulos, polígonos, círculo",
            "**6. Geometria Espacial** → Prismas, pirâmides, cone, cilindro, esfera",
            "**7. Geometria Analítica** → Ponto, reta, circunferência no plano cartesiano",
            "**8. Estatística e Probabilidade** → Média, mediana, moda, contagem, probabilidade",
            "**9. Progressões (PA e PG)** → Fórmulas e aplicações",
            "**10. Matemática Financeira** → Juros simples e compostos, porcentagem, descontos",
        ],
        "links": [
            ("▶️ Professor Ferretto — Matemática", "https://www.youtube.com/@ProfessorFerretto"),
            ("▶️ Matemática Rio — YouTube", "https://www.youtube.com/@matematicario"),
            ("🌐 Khan Academy — Matemática", "https://pt.khanacademy.org/math"),
            ("🌐 Toda Matéria — Matemática", "https://www.todamateria.com.br/matematica/"),
        ],
        "flashcards": [
            ("Qual é a fórmula de Bhaskara?", "x = (-b ± √Δ) / 2a, onde Δ = b² - 4ac. Usada para resolver equações do 2º grau ax² + bx + c = 0."),
            ("O que é probabilidade?", "P(A) = número de casos favoráveis / número de casos possíveis. Varia de 0 (impossível) a 1 (certo)."),
            ("O que é progressão aritmética (PA)?", "Sequência em que a diferença entre termos consecutivos é constante (razão r). Fórmula do termo geral: an = a1 + (n-1)·r"),
            ("Qual a fórmula de juros compostos?", "M = C · (1 + i)^t, onde M = montante, C = capital, i = taxa e t = tempo."),
        ],
    },
    "física": {
        "emoji": "🔬",
        "topicos": [
            "**1. Mecânica** → Cinemática, leis de Newton, trabalho, energia, impulso",
            "**2. Termologia** → Temperatura, calor, dilatação, calorimetria, termodinâmica",
            "**3. Óptica** → Reflexão, refração, lentes, espelhos",
            "**4. Ondas** → Som, luz, efeito Doppler, ondas eletromagnéticas",
            "**5. Eletricidade** → Circuitos, resistência, lei de Ohm, potência, corrente",
            "**6. Magnetismo** → Campo magnético, força magnética, indução eletromagnética",
            "**7. Física Moderna** → Relatividade, efeito fotoelétrico, radioatividade",
            "**8. Gravitação** → Leis de Kepler, força gravitacional, órbitas",
        ],
        "links": [
            ("▶️ Professor Marcelo Boaro — Física", "https://www.youtube.com/@MarceloBoaro"),
            ("▶️ Física e Vídeo — YouTube", "https://www.youtube.com/@FisicaeVideo"),
            ("🌐 Brasil Escola — Física", "https://brasilescola.uol.com.br/fisica"),
            ("🌐 Toda Matéria — Física", "https://www.todamateria.com.br/fisica/"),
        ],
        "flashcards": [
            ("Quais são as 3 Leis de Newton?", "1ª (Inércia): todo corpo tende a permanecer em repouso ou MRU. 2ª (F=ma): a força resulta em aceleração proporcional à massa. 3ª (Ação e Reação): toda ação gera reação igual e oposta."),
            ("O que é a Lei de Ohm?", "V = R · I, onde V é tensão (volts), R é resistência (ohms) e I é corrente elétrica (amperes)."),
            ("O que é velocidade escalar média?", "vm = Δs / Δt — variação do espaço dividida pela variação do tempo."),
            ("O que é energia cinética?", "Ec = (m · v²) / 2 — energia que um corpo possui devido ao seu movimento."),
        ],
    },
    "química": {
        "emoji": "⚗️",
        "topicos": [
            "**1. Estrutura Atômica** → Modelos atômicos, número atômico, distribuição eletrônica",
            "**2. Tabela Periódica** → Propriedades periódicas, famílias e períodos",
            "**3. Ligações Químicas** → Iônica, covalente, metálica, polaridade",
            "**4. Funções Inorgânicas** → Ácidos, bases, sais, óxidos",
            "**5. Reações Químicas** → Balanceamento, estequiometria, tipos de reações",
            "**6. Soluções** → Concentração, solubilidade, pH e pOH",
            "**7. Termoquímica e Cinética** → Entalpia, velocidade de reação, equilíbrio",
            "**8. Eletroquímica** → Pilhas, eletrólise, oxidação e redução",
            "**9. Química Orgânica** → Hidrocarbonetos, funções orgânicas, isomeria",
        ],
        "links": [
            ("▶️ Química com o Julio — YouTube", "https://www.youtube.com/@quimicacomojulio"),
            ("▶️ Khan Academy — Química", "https://pt.khanacademy.org/science/chemistry"),
            ("🌐 Brasil Escola — Química", "https://brasilescola.uol.com.br/quimica"),
            ("🌐 Toda Matéria — Química", "https://www.todamateria.com.br/quimica/"),
        ],
        "flashcards": [
            ("O que é pH?", "Potencial Hidrogeniônico — mede a acidez ou basicidade de uma solução. Varia de 0 a 14: abaixo de 7 é ácido, 7 é neutro, acima de 7 é básico."),
            ("O que é uma reação de oxirredução?", "Reação que envolve transferência de elétrons: o agente oxidante ganha elétrons (reduz) e o agente redutor perde elétrons (oxida)."),
            ("O que são isômeros?", "Compostos com a mesma fórmula molecular, mas estruturas diferentes, resultando em propriedades físicas ou químicas distintas."),
            ("O que é estequiometria?", "O cálculo das proporções entre reagentes e produtos em uma reação química balanceada, baseado nas leis de conservação de massa."),
        ],
    },
    "história": {
        "emoji": "📜",
        "topicos": [
            "**1. Brasil Colônia** → Descobrimento, ciclos econômicos, escravidão, resistência",
            "**2. Brasil Império** → Independência, período regencial, segundo reinado",
            "**3. Brasil República** → República Velha, Era Vargas, democracia, ditadura militar",
            "**4. Brasil Contemporâneo** → Redemocratização, Constituição de 1988",
            "**5. História Antiga e Medieval** → Grécia, Roma, feudalismo, Igreja Católica",
            "**6. Idade Moderna** → Renascimento, Reforma, absolutismo, mercantilismo",
            "**7. Revoluções** → Francesa, Industrial, Americana, Russa",
            "**8. Século XX** → 1ª e 2ª Guerra Mundial, Guerra Fria, descolonização",
            "**9. Mundo Contemporâneo** → Globalização, terrorismo, conflitos atuais",
        ],
        "links": [
            ("▶️ História do Mundo — YouTube", "https://www.youtube.com/@historiahoje"),
            ("▶️ Descomplica — História ENEM", "https://www.youtube.com/@descomplica"),
            ("🌐 Brasil Escola — História", "https://brasilescola.uol.com.br/historia"),
            ("🌐 Toda Matéria — História do Brasil", "https://www.todamateria.com.br/historia-do-brasil/"),
        ],
        "flashcards": [
            ("Qual foi o principal ciclo econômico do Brasil colonial?", "O país passou pelos ciclos do Pau-Brasil, Açúcar, Mineração (ouro e diamantes) e Café. O açúcar foi o mais duradouro, no século XVII."),
            ("O que foi a Revolução Francesa (1789)?", "Movimento que derrubou o Absolutismo na França, baseado nos ideais de Liberdade, Igualdade e Fraternidade. Originou a Declaração dos Direitos do Homem e do Cidadão."),
            ("O que foi a Guerra Fria?", "Conflito ideológico (1947-1991) entre EUA (capitalismo) e URSS (socialismo), marcado pela corrida armamentista, espacial e disputas por influência geopolítica, sem confronto direto."),
            ("O que foi o Estado Novo de Vargas?", "Ditadura de Getúlio Vargas (1937-1945), com centralização do poder, fechamento do Congresso, censura à imprensa e criação da CLT (consolidação das leis trabalhistas)."),
        ],
    },
    "geografia": {
        "emoji": "🌍",
        "topicos": [
            "**1. Cartografia** → Mapas, escalas, coordenadas, fusos horários, projeções",
            "**2. Geologia e Relevo** → Tectônica de placas, tipos de relevo, rochas, solos",
            "**3. Climatologia** → Tipos de clima, massas de ar, El Niño/La Niña, aquecimento global",
            "**4. Hidrografia** → Bacias hidrográficas, oceanos, crise hídrica",
            "**5. Biomas** → Amazônia, Cerrado, Caatinga, Mata Atlântica, Pampa, Pantanal",
            "**6. População** → Demografia, migrações, urbanização, transição demográfica",
            "**7. Geopolítica** → Blocos econômicos, conflitos territoriais, ONU",
            "**8. Economia** → Globalização, PIB, IDH, energia, agronegócio, indústria",
            "**9. Brasil** → Regiões, economia, desigualdade, problemas sociais, fronteiras",
        ],
        "links": [
            ("▶️ Geografia pelo Mundo — YouTube", "https://www.youtube.com/@geografiapelomundo"),
            ("▶️ Descomplica — Geografia ENEM", "https://www.youtube.com/@descomplica"),
            ("🌐 Brasil Escola — Geografia", "https://brasilescola.uol.com.br/geografia"),
            ("🌐 IBGE — Portal de Mapas", "https://portaldemapas.ibge.gov.br/portal.php"),
        ],
        "flashcards": [
            ("Quais são os 6 biomas brasileiros?", "Amazônia (maior, ~49% do território), Cerrado (savana mais rica do mundo), Caatinga (exclusivo do Brasil), Mata Atlântica (mais destruída), Pampa (sul) e Pantanal (maior área úmida do planeta)."),
            ("O que é IDH?", "Índice de Desenvolvimento Humano — mede a qualidade de vida combinando renda, educação e expectativa de vida. Vai de 0 a 1; quanto mais próximo de 1, melhor."),
            ("O que é El Niño?", "Fenômeno climático causado pelo aquecimento anormal das águas do Pacífico, que causa chuvas acima do normal no Sul do Brasil e seca no Nordeste."),
            ("O que são placas tectônicas?", "Grandes fragmentos da crosta terrestre em movimento constante. Seu choque e separação geram terremotos, vulcões e a formação de montanhas."),
        ],
    },
    "biologia": {
        "emoji": "🧬",
        "topicos": [
            "**1. Citologia** → Estrutura celular, organelas, membrana, divisão celular",
            "**2. Histologia** → Tecidos animais e vegetais, funções",
            "**3. Genética** → Leis de Mendel, DNA, RNA, mutações, biotecnologia",
            "**4. Evolução** → Teorias evolutivas, seleção natural, especiação, Darwin",
            "**5. Ecologia** → Ecossistemas, cadeias alimentares, ciclos biogeoquímicos",
            "**6. Fisiologia Humana** → Sistemas: nervoso, cardiovascular, digestório, respiratório",
            "**7. Botânica** → Classificação das plantas, fotossíntese, reprodução vegetal",
            "**8. Zoologia** → Classificação dos animais, características dos filos",
            "**9. Saúde** → Doenças infecciosas, vacinas, parasitoses, endemias",
        ],
        "links": [
            ("▶️ Professor Rubens Oda — Biologia", "https://www.youtube.com/@professorrubensoda"),
            ("▶️ Biologia Total — YouTube", "https://www.youtube.com/@biologiatotal"),
            ("🌐 Brasil Escola — Biologia", "https://brasilescola.uol.com.br/biologia"),
            ("🌐 Toda Matéria — Biologia", "https://www.todamateria.com.br/biologia/"),
        ],
        "flashcards": [
            ("Qual a diferença entre célula procarionte e eucarionte?", "Procariontes (bactérias) não possuem núcleo definido (DNA solto no citoplasma). Eucariontes (animais, plantas, fungos) têm núcleo delimitado por membrana nuclear."),
            ("O que são as leis de Mendel?", "1ª Lei (Segregação): cada caráter é determinado por par de fatores que se separam na formação dos gametas. 2ª Lei (Segregação Independente): genes de diferentes pares segregam-se independentemente."),
            ("O que é fotossíntese?", "Processo pelo qual plantas, algas e cianobactérias convertem luz solar, CO₂ e água em glicose (energia) e liberam O₂. Equação: 6CO₂ + 6H₂O + luz → C₆H₁₂O₆ + 6O₂."),
            ("O que é seleção natural?", "Mecanismo evolutivo proposto por Darwin: indivíduos com características mais adaptadas ao ambiente sobrevivem e se reproduzem mais, transmitindo essas características à descendência."),
        ],
    },
}

# ══════════════════════════════════════════════════════════════════
# SETUP DO BOT
# ══════════════════════════════════════════════════════════════════
intents = discord.Intents.default()
intents.message_content = True
intents.members = True

bot = commands.Bot(command_prefix="!", intents=intents)
ia_client = anthropic.Anthropic(api_key=ANTHROPIC_API_KEY)

# ══════════════════════════════════════════════════════════════════
# FUNÇÕES AUXILIARES
# ══════════════════════════════════════════════════════════════════

def sanitizar_entrada(texto: str) -> str | None:
    """Retorna None se detectar tentativa de injeção de prompt."""
    texto_lower = texto.lower()
    for termo in TERMOS_INJECAO:
        if termo in texto_lower:
            return None
    return texto


def limpar_json(texto: str) -> str:
    return texto.replace("```json", "").replace("```", "").strip()


def dividir_mensagem(texto: str, limite: int = 1900) -> list:
    if len(texto) <= limite:
        return [texto]
    blocos, resto = [], texto
    while resto:
        blocos.append(resto[:limite])
        resto = resto[limite:]
    return blocos


def detectar_disciplina(nome_canal: str) -> str | None:
    nome = nome_canal.lower()
    for disciplina in ASSUNTOS:
        if disciplina in nome:
            return disciplina
    return None


async def chamar_ia(
    system: str,
    user: str,
    max_tokens: int = 1000,
    guild_id: str = None,
    comando: str = "geral"
) -> str:
    """Chama a API da Anthropic com controle de saldo e registro de consumo."""
    if guild_id:
        garantir_servidor(guild_id)
        saldo = get_saldo(guild_id)
        if saldo <= 0:
            return "❌ O saldo de créditos de IA deste servidor acabou. Peça ao administrador para realizar uma recarga com `/adicionar-saldo`."

    try:
        response = ia_client.messages.create(
            model="claude-sonnet-4-20250514",
            max_tokens=max_tokens,
            system=system,
            messages=[{"role": "user", "content": user}]
        )

        if guild_id and hasattr(response, "usage"):
            it = response.usage.input_tokens
            ot = response.usage.output_tokens
            custo_usd = (it / 1_000_000 * PRECO_INPUT_USD) + (ot / 1_000_000 * PRECO_OUTPUT_USD)
            custo_brl = custo_usd * COTACAO_USD_BRL
            atualizar_saldo(guild_id, -custo_brl)
            registrar_consumo(guild_id, comando, it + ot, custo_brl)

        return response.content[0].text

    except Exception as e:
        print(f"[{_timestamp()}] 🔴 ERRO DA API (/{comando}): {e}")
        return f"❌ Erro na IA: {e}"


async def postar_assuntos_no_canal(canal: discord.TextChannel, disciplina: str):
    """Posta tópicos, links e flashcards de uma disciplina no canal."""
    dados = ASSUNTOS[disciplina]
    emoji = dados["emoji"]

    # ── Embed de Tópicos ──
    embed_topicos = discord.Embed(
        title=f"{emoji} Assuntos de {disciplina.capitalize()} — ENEM",
        description="Tópicos mais cobrados no ENEM. Use este canal para tirar dúvidas!",
        color=discord.Color.blurple()
    )
    topicos = dados["topicos"]
    metade = len(topicos) // 2
    embed_topicos.add_field(name="📋 Tópicos (Parte 1)", value="\n".join(topicos[:metade]), inline=False)
    if topicos[metade:]:
        embed_topicos.add_field(name="📋 Tópicos (Parte 2)", value="\n".join(topicos[metade:]), inline=False)
    embed_topicos.set_footer(text="📌 Fixado automaticamente pelo Willyam Bot")

    msg1 = await canal.send(embed=embed_topicos)
    await msg1.pin()

    # ── Embed de Links/Recursos ──
    if "links" in dados:
        embed_links = discord.Embed(
            title=f"🔗 Recursos Recomendados — {disciplina.capitalize()}",
            color=discord.Color.orange()
        )
        links_texto = "\n".join([f"[{nome}]({url})" for nome, url in dados["links"]])
        embed_links.add_field(name="📺 Vídeoaulas e Sites", value=links_texto, inline=False)
        embed_links.set_footer(text="Clique nos links para acessar os materiais")
        await canal.send(embed=embed_links)

    # ── Flashcards com Spoiler ──
    if "flashcards" in dados:
        flashcard_texto = f"## 🃏 Flashcards de {disciplina.capitalize()}\n*Clique nos spoilers para revelar a resposta!*\n\n"
        for i, (pergunta, resposta) in enumerate(dados["flashcards"], 1):
            flashcard_texto += (
                f"**🔹 Flashcard {i}:**\n"
                f"||**Pergunta:** {pergunta}\n"
                f"**Resposta:** {resposta}||\n\n"
            )
        await canal.send(flashcard_texto)

# ══════════════════════════════════════════════════════════════════
# EVENTO: BOT PRONTO
# ══════════════════════════════════════════════════════════════════

@bot.event
async def on_ready():
    init_db()
    print(f"[{_timestamp()}] ✅ Bot conectado como {bot.user}")
    print(f"[{_timestamp()}] 🖥️  Servidores: {len(bot.guilds)}")
    try:
        synced = await bot.tree.sync()
        print(f"[{_timestamp()}] ✅ {len(synced)} comandos slash sincronizados")
    except Exception as e:
        print(f"[{_timestamp()}] ❌ Erro ao sincronizar: {e}")
    tarefa_dica_diaria.start()

# ══════════════════════════════════════════════════════════════════
# EVENTO: MENSAGENS (Moderação + XP + IA Automática)
# ══════════════════════════════════════════════════════════════════

@bot.event
async def on_message(message: discord.Message):
    if message.author.bot:
        return

    conteudo = message.content.lower()

    # Moderação: palavras proibidas
    for palavra in PALAVRAS_PROIBIDAS:
        if palavra in conteudo:
            await message.delete()
            aviso = await message.channel.send(
                f"⚠️ {message.author.mention}, mensagem removida por conteúdo inapropriado. Leia as 📌regras!"
            )
            await aviso.delete(delay=8)
            return

    # XP automático por mensagem
    if message.guild:
        atualizar_xp(str(message.author.id), str(message.guild.id))

    # Canal de IA automática
    if message.channel.name in ("🤖pergunte-a-ia", "pergunte-a-ia"):
        if len(message.content) > 500:
            await message.reply("⚠️ Mensagem muito longa (máx. 500 caracteres). Resuma sua dúvida!")
            return

        entrada = sanitizar_entrada(message.content)
        if not entrada:
            await message.reply("🚫 Instrução inválida detectada por motivos de segurança.")
            return

        async with message.channel.typing():
            guild_id = str(message.guild.id) if message.guild else None
            resposta = await chamar_ia(
                system=(
                    "Você é um assistente educacional em um servidor do Discord focado em estudos para o ENEM. "
                    "Responda de forma clara, didática e motivadora. Use emojis com moderação. "
                    "Formate bem usando markdown do Discord (**negrito**, `código`, etc). "
                    "Seja conciso e objetivo, focando no que o ENEM cobra."
                ),
                user=message.content,
                guild_id=guild_id,
                comando="pergunte-a-ia"
            )
            for bloco in dividir_mensagem(resposta):
                await message.reply(bloco)
        return

    await bot.process_commands(message)

# ══════════════════════════════════════════════════════════════════
# EVENTO: MEMBRO ENTROU
# ══════════════════════════════════════════════════════════════════

@bot.event
async def on_member_join(member: discord.Member):
    cargo_novato = discord.utils.get(member.guild.roles, name="🆕 Novato")
    if cargo_novato:
        try:
            await member.add_roles(cargo_novato)
        except discord.Forbidden:
            pass

    canal = discord.utils.get(member.guild.text_channels, name="🙋apresentações") or \
            discord.utils.get(member.guild.text_channels, name="apresentações")
    if canal:
        embed = discord.Embed(
            title=f"👋 Bem-vindo(a), {member.display_name}!",
            description=(
                f"Olá {member.mention}! Seja bem-vindo(a) ao servidor de estudos! 📚\n\n"
                "📌 Leia as regras em **#📌regras**\n"
                "🙋 Apresente-se aqui!\n"
                "🤖 Use `/ia` para tirar dúvidas com IA a qualquer hora\n"
                "📚 Explore os canais das disciplinas com assuntos fixados\n"
                "🎯 Use `/meta` para definir seu objetivo no ENEM"
            ),
            color=discord.Color.green()
        )
        embed.set_thumbnail(url=member.display_avatar.url)
        embed.set_footer(text=f"Membro #{member.guild.member_count}")
        await canal.send(embed=embed)

# ══════════════════════════════════════════════════════════════════
# TAREFA AUTOMÁTICA: Dica Diária
# ══════════════════════════════════════════════════════════════════

@tasks.loop(hours=24)
async def tarefa_dica_diaria():
    """Envia uma dica de estudos diária automaticamente."""
    for guild in bot.guilds:
        canal = discord.utils.get(guild.text_channels, name="💬geral-estudos") or \
                discord.utils.get(guild.text_channels, name="geral-estudos")
        if canal:
            guild_id = str(guild.id)
            dica = await chamar_ia(
                system="Você é um coach de estudos para o ENEM.",
                user="Gere 1 dica motivacional e prática de estudos para o ENEM. Seja criativo, curto (máx. 3 linhas) e use 1 emoji.",
                max_tokens=150,
                guild_id=guild_id,
                comando="dica-automatica"
            )
            embed = discord.Embed(
                title="💡 Dica Diária de Estudos",
                description=dica,
                color=discord.Color.yellow()
            )
            embed.set_footer(text=f"📅 {datetime.now().strftime('%d/%m/%Y')} — Willyam Bot")
            try:
                await canal.send(embed=embed)
            except discord.Forbidden:
                pass

# ══════════════════════════════════════════════════════════════════
# TRATAMENTO GLOBAL DE ERROS
# ══════════════════════════════════════════════════════════════════

@bot.tree.error
async def on_app_command_error(interaction: discord.Interaction, error):
    if isinstance(error, app_commands.MissingPermissions):
        msg = "❌ Você não tem permissão para usar este comando."
    elif isinstance(error, app_commands.CommandOnCooldown):
        msg = f"⏳ Aguarde **{error.retry_after:.1f}s** antes de usar este comando novamente."
    elif isinstance(error, app_commands.CheckFailure):
        msg = f"🚫 {error}"
    else:
        msg = f"❌ Erro inesperado: {error}"
        print(f"[{_timestamp()}] ❌ Erro em comando: {error}")

    embed = discord.Embed(description=msg, color=discord.Color.red())
    try:
        await interaction.response.send_message(embed=embed, ephemeral=True)
    except Exception:
        try:
            await interaction.followup.send(embed=embed, ephemeral=True)
        except Exception:
            pass

# ══════════════════════════════════════════════════════════════════
# CHECK: Dono do Bot
# ══════════════════════════════════════════════════════════════════

def is_dono_bot():
    async def predicate(interaction: discord.Interaction) -> bool:
        if DONO_BOT_ID == 0:
            raise app_commands.CheckFailure("❌ DONO_BOT_ID não configurado no .env.")
        if interaction.user.id != DONO_BOT_ID:
            raise app_commands.CheckFailure("❌ Apenas o dono do bot pode usar este comando.")
        return True
    return app_commands.check(predicate)

# ══════════════════════════════════════════════════════════════════
# ════════════════ COMANDOS DO BOT ════════════════════════════════
# ══════════════════════════════════════════════════════════════════


# ─────────────────────────────────────────
# /organizar — Monta o servidor completo
# ─────────────────────────────────────────
@bot.tree.command(name="organizar", description="🏗️ Monta a estrutura completa do servidor de estudos")
@app_commands.checks.has_permissions(administrator=True)
async def organizar(interaction: discord.Interaction):
    await interaction.response.defer(ephemeral=True)
    guild = interaction.guild
    criados = 0
    erros = []

    # ── Cargos estruturais automáticos ──
    cargos_essenciais = [
        {"nome": "♂️ Homem",    "cor": discord.Color.from_rgb(52, 152, 219)},
        {"nome": "♀️ Mulher",   "cor": discord.Color.from_rgb(231, 76, 60)},
        {"nome": "🎓 Aluno",    "cor": discord.Color.green()},
        {"nome": "🧠 Monitor",  "cor": discord.Color.from_rgb(155, 89, 182)},
    ]
    for cargo_info in cargos_essenciais:
        if not discord.utils.get(guild.roles, name=cargo_info["nome"]):
            try:
                await guild.create_role(
                    name=cargo_info["nome"],
                    color=cargo_info["cor"],
                    mentionable=True
                )
            except discord.Forbidden:
                erros.append(f"Sem permissão para criar cargo: {cargo_info['nome']}")

    # ── Estrutura de canais ──
    for item in ESTRUTURA_SERVIDOR:
        try:
            cat = discord.utils.get(guild.categories, name=item["categoria"])
            if not cat:
                cat = await guild.create_category(item["categoria"])
        except discord.Forbidden:
            erros.append(f"Sem permissão para criar categoria: {item['categoria']}")
            continue

        for canal_info in item["canais"]:
            nome_busca = canal_info["nome"].replace(" ", "-").lower()
            existe = any(nome_busca in c.name.lower() for c in guild.channels)
            if existe:
                continue

            try:
                if canal_info["tipo"] == "voice":
                    await guild.create_voice_channel(canal_info["nome"], category=cat)
                else:
                    # Canal blindado (somente leitura para membros)
                    overwrites = {}
                    if canal_info.get("blindado"):
                        overwrites[guild.default_role] = discord.PermissionOverwrite(
                            read_messages=True,
                            send_messages=False
                        )

                    novo = await guild.create_text_channel(
                        canal_info["nome"],
                        category=cat,
                        overwrites=overwrites
                    )
                    disciplina = detectar_disciplina(canal_info["nome"])
                    if disciplina:
                        await postar_assuntos_no_canal(novo, disciplina)
                criados += 1
            except discord.Forbidden:
                erros.append(f"Sem permissão: {canal_info['nome']}")
            except discord.HTTPException as e:
                erros.append(f"Erro HTTP ({canal_info['nome']}): {e}")

    # ── Embed de resultado ──
    embed = discord.Embed(
        title="✅ Servidor Organizado com Sucesso!",
        color=discord.Color.green()
    )
    embed.add_field(name="📊 Canais Criados", value=str(criados), inline=True)
    embed.add_field(name="📌 Assuntos + Flashcards", value="Fixados automaticamente!", inline=True)
    if erros:
        embed.add_field(name="⚠️ Avisos", value="\n".join(erros[:5]), inline=False)
        embed.color = discord.Color.orange()
    embed.set_footer(text=f"Executado por {interaction.user.display_name}")
    await interaction.followup.send(embed=embed, ephemeral=True)


# ─────────────────────────────────────────
# /cargos — Cria todos os cargos
# ─────────────────────────────────────────
@bot.tree.command(name="cargos", description="🏷️ Cria os cargos do servidor de estudos")
@app_commands.checks.has_permissions(administrator=True)
async def criar_cargos(interaction: discord.Interaction):
    await interaction.response.defer(ephemeral=True)
    guild = interaction.guild
    criados = 0

    for cargo in CARGOS:
        if not discord.utils.get(guild.roles, name=cargo["nome"]):
            try:
                perms = discord.Permissions(administrator=True) if cargo["admin"] else discord.Permissions.none()
                await guild.create_role(
                    name=cargo["nome"],
                    color=cargo["cor"],
                    permissions=perms,
                    mentionable=True
                )
                criados += 1
            except discord.Forbidden:
                pass

    embed = discord.Embed(
        title=f"✅ {criados} Cargos Criados!",
        description="Todos os cargos do servidor foram configurados.",
        color=discord.Color.green()
    )
    await interaction.followup.send(embed=embed, ephemeral=True)


# ─────────────────────────────────────────
# /executar — Controle total por IA
# ─────────────────────────────────────────
@bot.tree.command(name="executar", description="🤖 Diga em português o que quer fazer e a IA executa")
@app_commands.describe(instrucao="Ex: crie um canal de redação, renomeie o canal X, apague 10 mensagens...")
@app_commands.checks.has_permissions(administrator=True)
@app_commands.checks.cooldown(1, 30, key=lambda i: i.guild_id)  # 1 uso a cada 30s por servidor
async def executar_comando(interaction: discord.Interaction, instrucao: str):
    await interaction.response.defer()

    if len(instrucao) > 500:
        await interaction.followup.send("⚠️ Instrução muito longa (máx. 500 caracteres).", ephemeral=True)
        return

    entrada = sanitizar_entrada(instrucao)
    if not entrada:
        embed = discord.Embed(description="🚫 Instrução inválida por motivos de segurança.", color=discord.Color.red())
        await interaction.followup.send(embed=embed, ephemeral=True)
        return

    guild    = interaction.guild
    guild_id = str(guild.id)

    system = """Você é o sistema de controle de um bot do Discord.
Receberá uma instrução em português e o estado atual do servidor.
Responda APENAS com um JSON válido descrevendo as ações a executar.
Formato:
{
  "resumo": "frase curta do que será feito",
  "acoes": [
    {"tipo": "criar_canal_texto", "nome": "nome-do-canal", "categoria": "NOME DA CATEGORIA ou null"},
    {"tipo": "criar_canal_voz",   "nome": "Nome do Canal", "categoria": "NOME DA CATEGORIA ou null"},
    {"tipo": "criar_categoria",   "nome": "NOME DA CATEGORIA"},
    {"tipo": "deletar_canal",     "nome": "nome-do-canal"},
    {"tipo": "renomear_canal",    "nome_atual": "nome-atual", "novo_nome": "novo-nome"},
    {"tipo": "criar_cargo",       "nome": "Nome do Cargo"},
    {"tipo": "mensagem",          "canal": "nome-do-canal", "conteudo": "texto da mensagem"},
    {"tipo": "limpar_canal",      "canal": "nome-do-canal", "quantidade": 10}
  ]
}
Nomes de canais: minúsculas, sem espaços (use hífen). Categorias em MAIÚSCULAS."""

    canais_texto = [c.name for c in guild.text_channels]
    canais_voz   = [c.name for c in guild.voice_channels]
    categorias   = [c.name for c in guild.categories]
    cargos       = [r.name for r in guild.roles if r.name != "@everyone"]

    user_prompt = f"""Instrução: {instrucao}
Estado atual:
- Categorias: {', '.join(categorias) or 'nenhuma'}
- Canais texto: {', '.join(canais_texto) or 'nenhum'}
- Canais voz: {', '.join(canais_voz) or 'nenhum'}
- Cargos: {', '.join(cargos) or 'nenhum'}"""

    texto = await chamar_ia(system=system, user=user_prompt, max_tokens=800, guild_id=guild_id, comando="executar")
    texto = limpar_json(texto)

    try:
        dados = json.loads(texto)
    except json.JSONDecodeError:
        await interaction.followup.send("❌ Não consegui interpretar a instrução. Tente ser mais específico.")
        return

    acoes  = dados.get("acoes", [])
    resumo = dados.get("resumo", "Executando...")
    log    = []

    for acao in acoes:
        tipo = acao.get("tipo")
        try:
            if tipo == "criar_categoria":
                nome = acao["nome"]
                if not discord.utils.get(guild.categories, name=nome):
                    await guild.create_category(nome)
                    log.append(f"✅ Categoria criada: **{nome}**")
                else:
                    log.append(f"⚠️ Já existe: **{nome}**")

            elif tipo == "criar_canal_texto":
                cat = discord.utils.get(guild.categories, name=acao.get("categoria"))
                await guild.create_text_channel(acao["nome"], category=cat)
                log.append(f"✅ Canal criado: **#{acao['nome']}**")

            elif tipo == "criar_canal_voz":
                cat = discord.utils.get(guild.categories, name=acao.get("categoria"))
                await guild.create_voice_channel(acao["nome"], category=cat)
                log.append(f"✅ Voz criado: **{acao['nome']}**")

            elif tipo == "deletar_canal":
                canal = discord.utils.get(guild.channels, name=acao["nome"])
                if canal:
                    await canal.delete()
                    log.append(f"🗑️ Deletado: **{acao['nome']}**")
                else:
                    log.append(f"⚠️ Não encontrado: **{acao['nome']}**")

            elif tipo == "renomear_canal":
                canal = discord.utils.get(guild.channels, name=acao["nome_atual"])
                if canal:
                    await canal.edit(name=acao["novo_nome"])
                    log.append(f"✏️ Renomeado: **{acao['nome_atual']}** → **{acao['novo_nome']}**")
                else:
                    log.append(f"⚠️ Não encontrado: **{acao['nome_atual']}**")

            elif tipo == "criar_cargo":
                nome = acao["nome"]
                if not discord.utils.get(guild.roles, name=nome):
                    await guild.create_role(name=nome, mentionable=True)
                    log.append(f"✅ Cargo criado: **{nome}**")
                else:
                    log.append(f"⚠️ Cargo já existe: **{nome}**")

            elif tipo == "mensagem":
                canal = discord.utils.get(guild.text_channels, name=acao["canal"])
                if canal:
                    await canal.send(acao["conteudo"])
                    log.append(f"💬 Mensagem enviada em **#{acao['canal']}**")
                else:
                    log.append(f"⚠️ Canal não encontrado: **{acao['canal']}**")

            elif tipo == "limpar_canal":
                canal = discord.utils.get(guild.text_channels, name=acao["canal"])
                if canal:
                    qtd = min(int(acao.get("quantidade", 10)), 100)
                    deletadas = await canal.purge(limit=qtd)
                    log.append(f"🗑️ {len(deletadas)} msgs removidas de **#{acao['canal']}**")
                else:
                    log.append(f"⚠️ Canal não encontrado: **{acao['canal']}**")

        except discord.Forbidden:
            log.append(f"🔒 Sem permissão para: `{tipo}`")
        except Exception as e:
            log.append(f"❌ Erro em `{tipo}`: {e}")

    embed = discord.Embed(
        title=f"🤖 {resumo}",
        description="\n".join(log) if log else "Nenhuma ação executada.",
        color=discord.Color.green() if log else discord.Color.orange()
    )
    embed.set_footer(text=f"Executado por {interaction.user.display_name}")
    await interaction.followup.send(embed=embed)


# ─────────────────────────────────────────
# /criar-categoria — IA monta pelo tema
# ─────────────────────────────────────────
@bot.tree.command(name="criar-categoria", description="🗂️ Descreva um tema e a IA cria a categoria com canais")
@app_commands.describe(tema="Ex: Redação ENEM, Grupo de Inglês, Física Quântica...")
@app_commands.checks.has_permissions(manage_channels=True)
async def criar_categoria(interaction: discord.Interaction, tema: str):
    await interaction.response.defer()
    guild_id = str(interaction.guild.id)

    prompt = f"""Sugira canais do Discord para a categoria sobre: "{tema}" (servidor de estudos ENEM).
Responda APENAS com JSON válido:
{{
  "categoria": "EMOJI NOME EM MAIÚSCULAS",
  "descricao": "frase curta descritiva",
  "canais": [
    {{"nome": "emoji-nome", "tipo": "text"}},
    {{"nome": "emoji-nome", "tipo": "voice"}}
  ]
}}
Regras: 4-6 canais, pelo menos 1 voz, nomes em minúsculas com hífen."""

    texto = await chamar_ia(
        system="Você monta estruturas de servidores Discord educacionais.",
        user=prompt,
        max_tokens=500,
        guild_id=guild_id,
        comando="criar-categoria"
    )
    texto = limpar_json(texto)

    try:
        dados = json.loads(texto)
        guild = interaction.guild
        nova_cat = await guild.create_category(dados["categoria"])
        canais_criados = []

        for canal in dados["canais"]:
            if canal["tipo"] == "voice":
                await guild.create_voice_channel(canal["nome"], category=nova_cat)
            else:
                novo = await guild.create_text_channel(canal["nome"], category=nova_cat)
                disciplina = detectar_disciplina(canal["nome"])
                if disciplina:
                    await postar_assuntos_no_canal(novo, disciplina)
            canais_criados.append(f"• {canal['nome']} ({'🔊' if canal['tipo']=='voice' else '💬'})")

        embed = discord.Embed(
            title=f"✅ {dados['categoria']}",
            description=dados.get("descricao", "Categoria criada com sucesso!"),
            color=discord.Color.green()
        )
        embed.add_field(name="Canais criados", value="\n".join(canais_criados))
        embed.set_footer(text=f"Criado por {interaction.user.display_name} com IA")
        await interaction.followup.send(embed=embed)

    except Exception as e:
        await interaction.followup.send(f"❌ Erro ao criar categoria: {e}")


# ─────────────────────────────────────────
# /assuntos — Posta assuntos no canal
# ─────────────────────────────────────────
@bot.tree.command(name="assuntos", description="📋 Posta e fixa os assuntos do ENEM no canal atual")
@app_commands.checks.has_permissions(manage_messages=True)
async def assuntos(interaction: discord.Interaction):
    await interaction.response.defer(ephemeral=True)
    guild_id   = str(interaction.guild.id)
    disciplina = detectar_disciplina(interaction.channel.name)

    if disciplina:
        await postar_assuntos_no_canal(interaction.channel, disciplina)
        await interaction.followup.send(
            f"✅ Assuntos de **{disciplina.capitalize()}** postados e fixados com flashcards!", ephemeral=True
        )
    else:
        resposta = await chamar_ia(
            system="Você é um professor especialista em ENEM.",
            user=f"Liste os principais assuntos cobrados no ENEM para o canal: {interaction.channel.name}. Formate em tópicos com **negrito**.",
            max_tokens=800,
            guild_id=guild_id,
            comando="assuntos"
        )
        embed = discord.Embed(
            title=f"📋 Assuntos — {interaction.channel.name}",
            description=resposta[:4000],
            color=discord.Color.blurple()
        )
        msg = await interaction.channel.send(embed=embed)
        await msg.pin()
        await interaction.followup.send("✅ Assuntos gerados pela IA e fixados!", ephemeral=True)


# ─────────────────────────────────────────
# /ia — Pergunta para a IA
# ─────────────────────────────────────────
@bot.tree.command(name="ia", description="🤖 Faça uma pergunta para a IA de estudos")
@app_commands.describe(pergunta="Sua dúvida ou pergunta sobre o ENEM")
@app_commands.checks.cooldown(1, 10, key=lambda i: i.user.id)  # 1 uso a cada 10s por usuário
async def ia_comando(interaction: discord.Interaction, pergunta: str):
    await interaction.response.defer(ephemeral=True)

    if len(pergunta) > 500:
        await interaction.followup.send("⚠️ Pergunta muito longa (máx. 500 caracteres).", ephemeral=True)
        return

    entrada = sanitizar_entrada(pergunta)
    if not entrada:
        embed = discord.Embed(description="🚫 Instrução inválida por motivos de segurança.", color=discord.Color.red())
        await interaction.followup.send(embed=embed, ephemeral=True)
        return

    guild_id = str(interaction.guild.id) if interaction.guild else None
    resposta = await chamar_ia(
        system=(
            "Você é um assistente educacional especializado no ENEM. "
            "Seja claro, didático e motivador. Use markdown do Discord para formatar. "
            "Responda em português brasileiro."
        ),
        user=pergunta,
        guild_id=guild_id,
        comando="ia"
    )
    embed = discord.Embed(
        title="🤖 Resposta da IA",
        description=resposta[:4000],
        color=discord.Color.blurple()
    )
    embed.set_footer(text=f"Pergunta de {interaction.user.display_name}")
    await interaction.followup.send(embed=embed, ephemeral=True)


# ─────────────────────────────────────────
# /resumo — Resumo de tópico com IA
# ─────────────────────────────────────────
@bot.tree.command(name="resumo", description="📝 Gera um resumo de qualquer tópico do ENEM com IA")
@app_commands.describe(topico="Ex: Revolução Francesa, Função Quadrática, Barroco...")
@app_commands.checks.cooldown(1, 15, key=lambda i: i.user.id)
async def resumo(interaction: discord.Interaction, topico: str):
    await interaction.response.defer(ephemeral=True)

    if len(topico) > 200:
        await interaction.followup.send("⚠️ Tópico muito longo (máx. 200 caracteres).", ephemeral=True)
        return

    guild_id = str(interaction.guild.id) if interaction.guild else None
    resposta = await chamar_ia(
        system="Você é um professor de cursinho pré-vestibular especialista em ENEM.",
        user=(
            f"Faça um resumo didático e completo sobre '{topico}' para o ENEM. "
            "Inclua: definição, pontos principais, o que cai no ENEM e 1 exemplo. "
            "Formate com emojis e **negrito** para facilitar a leitura."
        ),
        max_tokens=1200,
        guild_id=guild_id,
        comando="resumo"
    )
    embed = discord.Embed(
        title=f"📝 Resumo: {topico}",
        description=resposta[:4000],
        color=discord.Color.from_rgb(52, 152, 219)
    )
    embed.set_footer(text=f"Gerado por IA para {interaction.user.display_name}")
    await interaction.followup.send(embed=embed, ephemeral=True)


# ─────────────────────────────────────────
# /flashcards — Flashcards gerados pela IA
# ─────────────────────────────────────────
@bot.tree.command(name="flashcards", description="🃏 Gera flashcards interativos de qualquer tema com IA")
@app_commands.describe(tema="Ex: Leis de Newton, Romantismo, Fato Social...")
@app_commands.checks.cooldown(1, 20, key=lambda i: i.user.id)
async def flashcards_ia(interaction: discord.Interaction, tema: str):
    await interaction.response.defer(ephemeral=True)

    guild_id = str(interaction.guild.id) if interaction.guild else None
    resposta = await chamar_ia(
        system="Você é um professor especialista em criar materiais de estudo para o ENEM.",
        user=(
            f"Crie 5 flashcards de estudo sobre '{tema}' para o ENEM. "
            "Formato de cada flashcard:\n"
            "**🔹 Flashcard N:**\n"
            "||**Pergunta:** [pergunta]\n**Resposta:** [resposta completa e explicativa]||\n\n"
            "As perguntas devem ser estilo ENEM. Seja detalhado nas respostas."
        ),
        max_tokens=1500,
        guild_id=guild_id,
        comando="flashcards"
    )
    await interaction.followup.send(
        f"## 🃏 Flashcards: {tema}\n*Clique nos spoilers para revelar!*\n\n{resposta[:4000]}",
        ephemeral=True
    )


# ─────────────────────────────────────────
# /simulado — Questão estilo ENEM com IA
# ─────────────────────────────────────────
@bot.tree.command(name="simulado", description="📄 Gera uma questão estilo ENEM com gabarito")
@app_commands.describe(disciplina="Ex: matemática, história, biologia...")
@app_commands.checks.cooldown(1, 20, key=lambda i: i.user.id)
async def simulado(interaction: discord.Interaction, disciplina: str):
    await interaction.response.defer(ephemeral=True)

    guild_id = str(interaction.guild.id) if interaction.guild else None
    resposta = await chamar_ia(
        system="Você é um elaborador de questões para o ENEM (estilo INEP).",
        user=(
            f"Crie 1 questão estilo ENEM de {disciplina} com:\n"
            "- Enunciado (pode ter texto/contexto de apoio)\n"
            "- 5 alternativas (A a E)\n"
            "- Gabarito em spoiler: ||Resposta: LETRA - Justificativa||"
        ),
        max_tokens=800,
        guild_id=guild_id,
        comando="simulado"
    )
    embed = discord.Embed(
        title=f"📄 Questão ENEM — {disciplina.capitalize()}",
        description=resposta[:4000],
        color=discord.Color.from_rgb(142, 68, 173)
    )
    embed.set_footer(text="Clique no spoiler para ver o gabarito!")
    await interaction.followup.send(embed=embed, ephemeral=True)


# ─────────────────────────────────────────
# /redação — Avaliação ou estrutura de redação
# ─────────────────────────────────────────
@bot.tree.command(name="redação", description="✍️ Gera estrutura ou avalia sua redação ENEM com IA")
@app_commands.describe(
    tema="Tema da redação",
    texto="Cole seu texto para avaliação (opcional, deixe vazio para receber estrutura)"
)
@app_commands.checks.cooldown(1, 30, key=lambda i: i.user.id)
async def redacao_ia(interaction: discord.Interaction, tema: str, texto: str = None):
    await interaction.response.defer(ephemeral=True)

    guild_id = str(interaction.guild.id) if interaction.guild else None

    if texto:
        # Avaliar texto enviado
        if len(texto) > 3000:
            await interaction.followup.send("⚠️ Texto muito longo (máx. 3000 caracteres).", ephemeral=True)
            return
        prompt = (
            f"Avalie esta redação ENEM sobre '{tema}' nas 5 competências (nota 0-200 cada):\n\n"
            f"{texto}\n\n"
            "Dê uma nota estimada para cada competência, aponte pontos fortes, erros e sugestões de melhoria."
        )
        titulo = f"✍️ Avaliação: {tema}"
    else:
        # Gerar estrutura
        prompt = (
            f"Crie uma estrutura detalhada de redação ENEM sobre '{tema}' com:\n"
            "- Sugestão de introdução (com repertório)\n"
            "- 2 argumentos para o desenvolvimento com tópicos frasais\n"
            "- Proposta de intervenção completa com os 5 elementos\n"
            "- Dicas de conectivos para usar"
        )
        titulo = f"✍️ Estrutura de Redação: {tema}"

    resposta = await chamar_ia(
        system="Você é um professor especialista em redação dissertativa-argumentativa para o ENEM.",
        user=prompt,
        max_tokens=1500,
        guild_id=guild_id,
        comando="redacao"
    )
    embed = discord.Embed(title=titulo, description=resposta[:4000], color=discord.Color.from_rgb(39, 174, 96))
    embed.set_footer(text=f"Para {interaction.user.display_name} — Willyam Bot")
    await interaction.followup.send(embed=embed, ephemeral=True)


# ─────────────────────────────────────────
# /cronograma — Cronograma personalizado com IA
# ─────────────────────────────────────────
@bot.tree.command(name="cronograma", description="🗓️ Gera um cronograma de estudos personalizado para o ENEM")
@app_commands.describe(
    horas_por_dia="Horas disponíveis para estudar por dia (ex: 3)",
    dias="Dias disponíveis por semana (ex: 5)",
    foco="Matéria(s) que precisa reforçar (ex: matemática e física)"
)
@app_commands.checks.cooldown(1, 30, key=lambda i: i.user.id)
async def cronograma(interaction: discord.Interaction, horas_por_dia: int, dias: int, foco: str = "todas"):
    await interaction.response.defer(ephemeral=True)

    guild_id = str(interaction.guild.id) if interaction.guild else None
    resposta = await chamar_ia(
        system="Você é um coach de estudos especializado em preparação para o ENEM.",
        user=(
            f"Crie um cronograma semanal de estudos para o ENEM com:\n"
            f"- {horas_por_dia}h por dia\n"
            f"- {dias} dias por semana\n"
            f"- Foco em: {foco}\n"
            "Inclua todas as disciplinas do ENEM, com tempo proporcional. "
            "Formate em tabela ou lista organizada. Adicione dicas de estudo."
        ),
        max_tokens=1200,
        guild_id=guild_id,
        comando="cronograma"
    )
    embed = discord.Embed(
        title="🗓️ Seu Cronograma de Estudos ENEM",
        description=resposta[:4000],
        color=discord.Color.from_rgb(241, 196, 15)
    )
    embed.set_footer(text=f"Gerado para {interaction.user.display_name}")
    await interaction.followup.send(embed=embed, ephemeral=True)


# ─────────────────────────────────────────
# /dica-enem — Dica aleatória de estudos
# ─────────────────────────────────────────
@bot.tree.command(name="dica-enem", description="💡 Receba uma dica de estudos para o ENEM")
@app_commands.checks.cooldown(1, 30, key=lambda i: i.user.id)
async def dica_enem(interaction: discord.Interaction):
    await interaction.response.defer(ephemeral=True)

    guild_id = str(interaction.guild.id) if interaction.guild else None
    disciplinas = list(ASSUNTOS.keys())
    disciplina_aleatoria = random.choice(disciplinas)

    resposta = await chamar_ia(
        system="Você é um professor de cursinho motivador e especialista no ENEM.",
        user=(
            f"Dê 1 dica prática e motivacional sobre como estudar {disciplina_aleatoria} para o ENEM. "
            "Inclua: o que mais cai, como estudar melhor e uma frase motivacional. Seja direto (máx. 5 linhas)."
        ),
        max_tokens=300,
        guild_id=guild_id,
        comando="dica-enem"
    )
    embed = discord.Embed(
        title=f"💡 Dica de {disciplina_aleatoria.capitalize()}",
        description=resposta,
        color=discord.Color.yellow()
    )
    embed.set_footer(text="Use /dica-enem para mais dicas!")
    await interaction.followup.send(embed=embed, ephemeral=True)


# ─────────────────────────────────────────
# /calcular — Calcula nota estimada do ENEM
# ─────────────────────────────────────────
@bot.tree.command(name="calcular", description="🔢 Calcula sua nota estimada no ENEM (TRI simplificado)")
@app_commands.describe(
    acertos_lc="Acertos em Linguagens (0-45)",
    acertos_ch="Acertos em Ciências Humanas (0-45)",
    acertos_cn="Acertos em Ciências da Natureza (0-45)",
    acertos_mt="Acertos em Matemática (0-45)",
    acertos_redacao="Nota da redação (0-1000)"
)
async def calcular_nota(
    interaction: discord.Interaction,
    acertos_lc: int,
    acertos_ch: int,
    acertos_cn: int,
    acertos_mt: int,
    acertos_redacao: int
):
    await interaction.response.defer(ephemeral=True)

    # Estimativa simplificada (não é TRI real, mas serve para referência)
    def estimar_nota(acertos: int, total: int = 45) -> float:
        pct = acertos / total
        return round(300 + pct * 600, 1)  # escala 300-900

    nota_lc  = estimar_nota(acertos_lc)
    nota_ch  = estimar_nota(acertos_ch)
    nota_cn  = estimar_nota(acertos_cn)
    nota_mt  = estimar_nota(acertos_mt)
    nota_red = min(max(acertos_redacao, 0), 1000)
    media    = round((nota_lc + nota_ch + nota_cn + nota_mt + nota_red) / 5, 1)

    cor = discord.Color.green() if media >= 700 else (discord.Color.yellow() if media >= 550 else discord.Color.red())
    embed = discord.Embed(title="📊 Estimativa de Nota ENEM", color=cor)
    embed.add_field(name="📖 Linguagens",       value=f"**{nota_lc}** pts ({acertos_lc}/45)", inline=True)
    embed.add_field(name="🌍 Ciências Humanas", value=f"**{nota_ch}** pts ({acertos_ch}/45)", inline=True)
    embed.add_field(name="🧬 Ciências Natureza",value=f"**{nota_cn}** pts ({acertos_cn}/45)", inline=True)
    embed.add_field(name="🔢 Matemática",       value=f"**{nota_mt}** pts ({acertos_mt}/45)", inline=True)
    embed.add_field(name="✍️ Redação",           value=f"**{nota_red}** pts",                  inline=True)
    embed.add_field(name="🏆 Média Geral",       value=f"**{media}** pts",                     inline=True)
    embed.add_field(
        name="📈 Classificação",
        value=(
            "🥇 Excelente!" if media >= 750 else
            "🥈 Muito bom!" if media >= 650 else
            "🥉 Bom, continue estudando!" if media >= 550 else
            "📚 Precisa reforçar mais!"
        ),
        inline=False
    )
    embed.set_footer(text="⚠️ Esta é uma estimativa simplificada. O TRI real pode variar.")
    await interaction.followup.send(embed=embed, ephemeral=True)


# ─────────────────────────────────────────
# /limpar — Apaga mensagens
# ─────────────────────────────────────────
@bot.tree.command(name="limpar", description="🗑️ Apaga mensagens do canal (máx. 100)")
@app_commands.describe(quantidade="Quantidade de mensagens para apagar")
@app_commands.checks.has_permissions(manage_messages=True)
async def limpar(interaction: discord.Interaction, quantidade: int):
    if not 1 <= quantidade <= 100:
        await interaction.response.send_message("❌ Escolha entre 1 e 100.", ephemeral=True)
        return
    await interaction.response.defer(ephemeral=True)
    deletadas = await interaction.channel.purge(limit=quantidade)
    embed = discord.Embed(
        description=f"🗑️ **{len(deletadas)}** mensagens apagadas com sucesso!",
        color=discord.Color.green()
    )
    await interaction.followup.send(embed=embed, ephemeral=True)


# ─────────────────────────────────────────
# /avisar — Avisa um usuário
# ─────────────────────────────────────────
@bot.tree.command(name="avisar", description="⚠️ Envia um aviso oficial para um usuário")
@app_commands.describe(usuario="Usuário a avisar", motivo="Motivo do aviso")
@app_commands.checks.has_permissions(manage_messages=True)
async def avisar(interaction: discord.Interaction, usuario: discord.Member, motivo: str):
    guild_id = str(interaction.guild.id)
    registrar_aviso(guild_id, str(usuario.id), motivo, str(interaction.user.id))

    avisos = get_avisos(guild_id, str(usuario.id))
    embed = discord.Embed(title="⚠️ Aviso Oficial", color=discord.Color.orange())
    embed.add_field(name="Usuário",    value=usuario.mention,         inline=True)
    embed.add_field(name="Motivo",     value=motivo,                  inline=True)
    embed.add_field(name="Moderador",  value=interaction.user.mention,inline=True)
    embed.add_field(name="Total Avisos", value=f"{len(avisos)} aviso(s)", inline=False)
    embed.set_footer(text="Leia as regras do servidor.")
    await interaction.channel.send(embed=embed)
    await interaction.response.send_message(f"✅ Aviso enviado para **{usuario.display_name}**.", ephemeral=True)


# ─────────────────────────────────────────
# /ver-avisos — Histórico de avisos
# ─────────────────────────────────────────
@bot.tree.command(name="ver-avisos", description="📋 Veja o histórico de avisos de um usuário")
@app_commands.describe(usuario="Usuário para consultar")
@app_commands.checks.has_permissions(manage_messages=True)
async def ver_avisos(interaction: discord.Interaction, usuario: discord.Member):
    avisos = get_avisos(str(interaction.guild.id), str(usuario.id))
    embed = discord.Embed(
        title=f"📋 Histórico de Avisos — {usuario.display_name}",
        color=discord.Color.orange() if avisos else discord.Color.green()
    )
    if avisos:
        for i, (motivo, moderador_id, data) in enumerate(avisos, 1):
            embed.add_field(
                name=f"Aviso {i} — {data[:10]}",
                value=f"Motivo: {motivo}\nMod: <@{moderador_id}>",
                inline=False
            )
    else:
        embed.description = "✅ Nenhum aviso encontrado para este usuário."
    await interaction.response.send_message(embed=embed, ephemeral=True)


# ─────────────────────────────────────────
# /regras — Posta as regras
# ─────────────────────────────────────────
@bot.tree.command(name="regras", description="📋 Posta as regras do servidor")
@app_commands.checks.has_permissions(manage_messages=True)
async def regras(interaction: discord.Interaction):
    embed = discord.Embed(
        title="📋 Regras do Servidor — Leia com atenção!",
        color=discord.Color.blue()
    )
    itens = [
        ("1️⃣ Respeito mútuo",         "Sem ofensas, discriminação, bullying ou preconceito de qualquer tipo."),
        ("2️⃣ Foco nos estudos",        "Mantenha os assuntos nos canais corretos. Use cada canal para seu propósito."),
        ("3️⃣ Sem spam ou flood",       "Evite mensagens repetidas, caps-lock excessivo ou muitas mensagens seguidas."),
        ("4️⃣ Português nos gerais",    "Use português nos canais gerais. Inglês e espanhol apenas nos canais específicos."),
        ("5️⃣ Colaboração",             "Compartilhe materiais, responda dúvidas dos colegas e ajude a comunidade crescer!"),
        ("6️⃣ IA com responsabilidade", "Use a IA para dúvidas de estudo. Não tente manipular ou abusar dos comandos."),
        ("7️⃣ Sem NSFW",               "Conteúdo adulto ou inapropriado é estritamente proibido."),
        ("8️⃣ Moderação",              "Sistema de punições: ⚠️ Aviso → 🔇 Silêncio → 🔨 Ban permanente."),
    ]
    for nome, valor in itens:
        embed.add_field(name=nome, value=valor, inline=False)
    embed.set_footer(text="📚 Bons estudos para todos! — Willyam Bot")
    await interaction.channel.send(embed=embed)
    await interaction.response.send_message("✅ Regras postadas!", ephemeral=True)


# ─────────────────────────────────────────
# /setup-boas-vindas — Configura boas-vindas
# ─────────────────────────────────────────
@bot.tree.command(name="setup-boas-vindas", description="👋 Envia a mensagem de boas-vindas no canal atual")
@app_commands.checks.has_permissions(administrator=True)
async def setup_boas_vindas(interaction: discord.Interaction):
    embed = discord.Embed(
        title="👋 Bem-vindo(a) ao Servidor de Estudos!",
        description=(
            "Este é um espaço dedicado aos estudos para o **ENEM** e muito mais!\n\n"
            "**Como começar:**\n"
            "1️⃣ Leia as regras em <#regras>\n"
            "2️⃣ Apresente-se no canal de apresentações\n"
            "3️⃣ Use `/meta` para definir seu objetivo\n"
            "4️⃣ Explore os canais de disciplinas\n"
            "5️⃣ Use `/ia` para tirar dúvidas a qualquer hora!\n\n"
            "**Comandos úteis:**\n"
            "`/ia` → Pergunte qualquer coisa para a IA\n"
            "`/simulado` → Questões estilo ENEM\n"
            "`/flashcards` → Estude com flashcards\n"
            "`/resumo` → Resumo de qualquer tópico\n"
            "`/cronograma` → Seu plano de estudos personalizado\n"
            "`/calcular` → Estime sua nota no ENEM"
        ),
        color=discord.Color.green()
    )
    embed.set_footer(text="📚 Willyam Bot — Seu assistente de estudos")
    await interaction.channel.send(embed=embed)
    await interaction.response.send_message("✅ Mensagem de boas-vindas enviada!", ephemeral=True)


# ─────────────────────────────────────────
# /info — Informações do servidor
# ─────────────────────────────────────────
@bot.tree.command(name="info", description="ℹ️ Informações sobre o servidor")
async def info(interaction: discord.Interaction):
    guild = interaction.guild
    embed = discord.Embed(title=f"ℹ️ {guild.name}", color=discord.Color.blurple())
    embed.add_field(name="👥 Membros",      value=str(guild.member_count),      inline=True)
    embed.add_field(name="💬 Canais Texto", value=str(len(guild.text_channels)), inline=True)
    embed.add_field(name="🔊 Canais Voz",  value=str(len(guild.voice_channels)),inline=True)
    embed.add_field(name="🏷️ Cargos",      value=str(len(guild.roles) - 1),    inline=True)
    embed.add_field(name="📅 Criado em",   value=guild.created_at.strftime("%d/%m/%Y"), inline=True)
    embed.add_field(name="👑 Dono",        value=guild.owner.mention if guild.owner else "N/A", inline=True)
    if guild.icon:
        embed.set_thumbnail(url=guild.icon.url)
    embed.set_footer(text="Willyam Bot — Servidor de Estudos")
    await interaction.response.send_message(embed=embed)


# ─────────────────────────────────────────
# /perfil — Perfil de estudos do usuário
# ─────────────────────────────────────────
@bot.tree.command(name="perfil", description="👤 Veja seu perfil de estudos")
@app_commands.describe(usuario="Usuário para ver o perfil (opcional)")
async def perfil(interaction: discord.Interaction, usuario: discord.Member = None):
    alvo = usuario or interaction.user
    guild_id = str(interaction.guild.id)
    dados = get_perfil(str(alvo.id), guild_id)

    embed = discord.Embed(title=f"👤 Perfil — {alvo.display_name}", color=discord.Color.blurple())
    embed.set_thumbnail(url=alvo.display_avatar.url)
    embed.add_field(name="🎯 Meta",       value=dados["meta"] or "Não definida", inline=False)
    embed.add_field(name="⭐ XP",         value=f"{dados['xp']} pontos",         inline=True)
    embed.add_field(name="💬 Mensagens",  value=str(dados["mensagens"]),          inline=True)
    nivel = dados["xp"] // 100 + 1
    embed.add_field(name="🏆 Nível",      value=f"Nível {nivel}",                 inline=True)
    embed.set_footer(text="Use /meta para definir seu objetivo!")
    await interaction.response.send_message(embed=embed)


# ─────────────────────────────────────────
# /meta — Define meta de estudos
# ─────────────────────────────────────────
@bot.tree.command(name="meta", description="🎯 Define sua meta de estudos no servidor")
@app_commands.describe(objetivo="Ex: Passar em medicina na USP, nota 900, entrar na federal...")
async def meta(interaction: discord.Interaction, objetivo: str):
    if len(objetivo) > 200:
        await interaction.response.send_message("⚠️ Meta muito longa (máx. 200 caracteres).", ephemeral=True)
        return
    definir_meta(str(interaction.user.id), str(interaction.guild.id), objetivo)
    embed = discord.Embed(
        title="🎯 Meta Definida!",
        description=f"**{interaction.user.display_name}**, sua meta foi salva:\n\n> *{objetivo}*",
        color=discord.Color.green()
    )
    embed.set_footer(text="Continue firme! Use /perfil para acompanhar seu progresso.")
    await interaction.response.send_message(embed=embed)


# ─────────────────────────────────────────
# /ranking — Ranking do servidor
# ─────────────────────────────────────────
@bot.tree.command(name="ranking", description="🏆 Veja o ranking de participação do servidor")
async def ranking(interaction: discord.Interaction):
    guild_id = str(interaction.guild.id)
    dados = get_ranking(guild_id)

    embed = discord.Embed(title="🏆 Ranking do Servidor", color=discord.Color.gold())
    medalhas = ["🥇", "🥈", "🥉"] + ["4️⃣", "5️⃣", "6️⃣", "7️⃣", "8️⃣", "9️⃣", "🔟"]

    if dados:
        for i, (user_id, xp, mensagens) in enumerate(dados):
            nivel = xp // 100 + 1
            embed.add_field(
                name=f"{medalhas[i]} <@{user_id}>",
                value=f"⭐ {xp} XP | 💬 {mensagens} msgs | 🏆 Nível {nivel}",
                inline=False
            )
    else:
        embed.description = "Nenhum dado encontrado ainda. Comecem a participar!"

    embed.set_footer(text="XP é ganho enviando mensagens. Continue ativo!")
    await interaction.response.send_message(embed=embed)


# ─────────────────────────────────────────
# /sorteio — Sorteia um membro
# ─────────────────────────────────────────
@bot.tree.command(name="sorteio", description="🎲 Sorteia um membro do servidor")
@app_commands.describe(motivo="Motivo do sorteio (ex: Ganhador do quiz de história)")
@app_commands.checks.has_permissions(manage_messages=True)
async def sorteio(interaction: discord.Interaction, motivo: str = "Sorteio"):
    membros = [m for m in interaction.guild.members if not m.bot]
    if not membros:
        await interaction.response.send_message("❌ Nenhum membro elegível.", ephemeral=True)
        return

    vencedor = random.choice(membros)
    embed = discord.Embed(
        title="🎲 Resultado do Sorteio!",
        description=f"## 🎉 {vencedor.mention} foi sorteado(a)!",
        color=discord.Color.gold()
    )
    embed.add_field(name="🏷️ Motivo",       value=motivo,                        inline=True)
    embed.add_field(name="👥 Participantes", value=str(len(membros)),             inline=True)
    embed.add_field(name="🎰 Sorteado por", value=interaction.user.mention,      inline=True)
    embed.set_thumbnail(url=vencedor.display_avatar.url)
    await interaction.response.send_message(embed=embed)


# ══════════════════════════════════════════════════════════════════
# COMANDOS DE CRÉDITOS — PÚBLICO
# ══════════════════════════════════════════════════════════════════

@bot.tree.command(name="saldo", description="💰 Veja os créditos de IA disponíveis neste servidor")
async def saldo(interaction: discord.Interaction):
    guild_id = str(interaction.guild.id)
    garantir_servidor(guild_id)
    saldo_atual = get_saldo(guild_id)
    total_tok, total_custo = get_total_tokens(guild_id)

    cor = discord.Color.green() if saldo_atual > 1 else discord.Color.red()
    embed = discord.Embed(title="💰 Créditos de IA do Servidor", color=cor)
    embed.add_field(name="💵 Saldo Atual",       value=f"R$ {saldo_atual:.4f}",  inline=True)
    embed.add_field(name="🔢 Tokens Gastos",      value=f"{total_tok:,}",         inline=True)
    embed.add_field(name="💸 Total Gasto (R$)",   value=f"R$ {total_custo:.4f}", inline=True)
    embed.add_field(
        name="📊 Status",
        value="✅ Saldo OK" if saldo_atual > 0.50 else "⚠️ Saldo baixo — solicite recarga!" if saldo_atual > 0 else "❌ Sem saldo",
        inline=False
    )
    embed.set_footer(text=f"Servidor: {interaction.guild.name}")
    await interaction.response.send_message(embed=embed, ephemeral=True)


# ══════════════════════════════════════════════════════════════════
# COMANDOS DE CRÉDITOS — DONO DO BOT
# ══════════════════════════════════════════════════════════════════

@bot.tree.command(name="adicionar-saldo", description="[DONO] Adiciona créditos em Reais a um servidor")
@app_commands.describe(guild_id="ID do servidor", valor="Valor em Reais a adicionar")
@is_dono_bot()
async def adicionar_saldo(interaction: discord.Interaction, guild_id: str, valor: float):
    # Validar se o bot está no servidor
    guild_alvo = bot.get_guild(int(guild_id)) if guild_id.isdigit() else None
    if not guild_alvo:
        embed = discord.Embed(description=f"❌ Servidor `{guild_id}` não encontrado ou bot não está presente.", color=discord.Color.red())
        await interaction.response.send_message(embed=embed, ephemeral=True)
        return

    garantir_servidor(guild_id)
    atualizar_saldo(guild_id, valor)
    novo_saldo = get_saldo(guild_id)

    embed = discord.Embed(title="✅ Saldo Adicionado", color=discord.Color.green())
    embed.add_field(name="🖥️ Servidor",   value=f"{guild_alvo.name} ({guild_id})", inline=False)
    embed.add_field(name="➕ Adicionado",  value=f"R$ {valor:.2f}",                inline=True)
    embed.add_field(name="💰 Novo Saldo", value=f"R$ {novo_saldo:.4f}",           inline=True)
    await interaction.response.send_message(embed=embed, ephemeral=True)


@bot.tree.command(name="remover-saldo", description="[DONO] Remove créditos de um servidor")
@app_commands.describe(guild_id="ID do servidor", valor="Valor a remover (0 = zerar tudo)")
@is_dono_bot()
async def remover_saldo(interaction: discord.Interaction, guild_id: str, valor: float):
    guild_alvo = bot.get_guild(int(guild_id)) if guild_id.isdigit() else None
    if not guild_alvo:
        embed = discord.Embed(description=f"❌ Servidor `{guild_id}` não encontrado.", color=discord.Color.red())
        await interaction.response.send_message(embed=embed, ephemeral=True)
        return

    garantir_servidor(guild_id)
    if valor == 0:
        try:
            with sqlite3.connect("banco.db") as conn:
                conn.execute("UPDATE servidores SET saldo_reais = 0 WHERE guild_id = ?", (guild_id,))
                conn.commit()
        except sqlite3.Error as e:
            await interaction.response.send_message(f"❌ Erro DB: {e}", ephemeral=True)
            return
        await interaction.response.send_message(f"✅ Saldo do servidor `{guild_alvo.name}` zerado.", ephemeral=True)
    else:
        atualizar_saldo(guild_id, -valor)
        novo_saldo = get_saldo(guild_id)
        embed = discord.Embed(title="✅ Saldo Removido", color=discord.Color.orange())
        embed.add_field(name="🖥️ Servidor",   value=guild_alvo.name,   inline=True)
        embed.add_field(name="➖ Removido",    value=f"R$ {valor:.2f}", inline=True)
        embed.add_field(name="💰 Novo Saldo", value=f"R$ {novo_saldo:.4f}", inline=True)
        await interaction.response.send_message(embed=embed, ephemeral=True)


@bot.tree.command(name="consultar-servidor", description="[DONO] Relatório completo de um servidor")
@app_commands.describe(guild_id="ID do servidor")
@is_dono_bot()
async def consultar_servidor(interaction: discord.Interaction, guild_id: str):
    garantir_servidor(guild_id)
    guild_alvo = bot.get_guild(int(guild_id)) if guild_id.isdigit() else None
    saldo_atual = get_saldo(guild_id)
    total_tok, total_custo = get_total_tokens(guild_id)

    embed = discord.Embed(title="📊 Relatório do Servidor", color=discord.Color.blurple())
    embed.add_field(name="🖥️ Nome",             value=guild_alvo.name if guild_alvo else "Desconhecido", inline=True)
    embed.add_field(name="🆔 Guild ID",          value=guild_id,               inline=True)
    embed.add_field(name="👥 Membros",           value=str(guild_alvo.member_count) if guild_alvo else "N/A", inline=True)
    embed.add_field(name="💰 Saldo Atual",       value=f"R$ {saldo_atual:.4f}", inline=True)
    embed.add_field(name="🔢 Tokens Gastos",     value=f"{total_tok:,}",        inline=True)
    embed.add_field(name="💸 Total Gasto (R$)", value=f"R$ {total_custo:.4f}", inline=True)
    await interaction.response.send_message(embed=embed, ephemeral=True)


# ══════════════════════════════════════════════════════════════════
# INICIALIZAÇÃO
# ══════════════════════════════════════════════════════════════════

if __name__ == "__main__":
    init_db()
    print(f"[{_timestamp()}] 🚀 Iniciando Willyam Bot...")
    bot.run(DISCORD_TOKEN)
