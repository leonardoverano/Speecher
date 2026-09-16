<p align="center"><img src="docs/logo.png" width="128" alt="Speecher"></p>
<h1 align="center">Speecher</h1>
<p align="center"><b>Segure uma tecla, fale, solte — o texto sai pronto onde o cursor estiver.</b><br>
Ditado inteligente para Windows: transcrição + limpeza + formatação por perfil, em ~2s e R$ 0/mês.</p>
<p align="center">
  <a href="../../releases"><img src="https://img.shields.io/badge/release-latest-blue" alt="release"></a>
  <img src="https://img.shields.io/badge/plataforma-Windows%2011-blue" alt="Windows 11">
</p>

Segure **Ctrl+Win**, fale, solte. O que você disse é transcrito, limpo (sem hesitações e
autocorreções da fala) e formatado no estilo do perfil ativo — e-mail, jurídico, WhatsApp,
roteiro — direto no campo onde o cursor estiver.

Exemplo real do comportamento: ditar *"Certifico que fui até o endereço, é, não, melhor,
dirigi-me ao endereço indicado no mandado... deixa eu corrigir, o imóvel pertence ao pai da
parte..."* produz *"Certifico que me dirigi ao endereço indicado no mandado. No local, fui
informado de que o imóvel pertence ao genitor da parte..."*.

<p align="center"><img src="docs/img/ui.png" width="640" alt="Janela de configurações"></p>
<p align="center"><img src="docs/img/overlay.png" width="400" alt="Overlay de ondas durante o ditado"><br>
<i>Enquanto você fala, uma onda discreta aparece na parte inferior da tela.</i></p>

## Instalação (usuários)

Não precisa saber programar. Você vai precisar de duas chaves gratuitas (5 minutos, sem cartão
de crédito) e do instalador.

**1. Baixe e instale**

- Vá em [Releases](../../releases) e baixe o `Speecher_x.y.z_x64-setup.exe` mais recente da lista
- Execute. O Windows SmartScreen vai avisar que o app não é reconhecido (ele não tem assinatura
  digital paga): clique em **"Mais informações" → "Executar assim mesmo"**. O código-fonte
  completo está neste repositório para quem quiser auditar.
- Ao final, o Speecher aparece como um ícone na bandeja do sistema (perto do relógio)

**2. Crie as duas chaves gratuitas**

| Chave | Onde criar | Para quê |
|---|---|---|
| Groq | [console.groq.com/keys](https://console.groq.com/keys) → "Create API Key" | Transcrever sua voz (~2.000 ditados grátis/dia) |
| Gemini | [aistudio.google.com/apikey](https://aistudio.google.com/apikey) → "Create API key" | Limpar e formatar o texto (free tier) |

Ambas usam login Google e não pedem cartão.

**3. Configure (uma vez)**

- Clique no ícone do Speecher na bandeja → abre a janela de configurações
- Na aba **Geral**, cole a chave Groq no campo "Chave da API Groq" e a chave Gemini no campo
  "Chave da API Gemini" (o ícone de olho revela o que você colou)
- Escolha seu microfone, se não quiser o padrão do sistema

**4. Use**

Clique em qualquer campo de texto (e-mail, WhatsApp Web, Word...), **segure Ctrl+Win, fale, e
solte**. Uma onda discreta aparece no rodapé da tela enquanto você fala; ~2 segundos depois de
soltar, o texto limpo aparece onde o cursor estava. Na aba **Perfis** você escolhe o estilo do
texto (natural, e-mail, jurídico formal, WhatsApp curto, roteiro).

Notas:
- **Privacidade**: o áudio vai para a Groq e o texto para o Google (free tiers podem usar dados
  para treino). Para transcrição 100% local/offline, veja "Modo local" abaixo — exige GPU NVIDIA
  e Python.
- O app se registra para **iniciar com o sistema** automaticamente na primeira execução
  (dá para desligar em Configurações → Geral).
- Sem a chave Gemini o texto sai bruto (sem limpeza); sem a chave Groq (e sem modo local) o
  ditado não funciona.

### Modo local (opcional, para quem tem GPU NVIDIA)

A transcrição pode rodar 100% na sua máquina (o áudio nunca sai do PC), como fallback automático
ou como provedor principal:

```powershell
git clone <URL-DO-REPOSITORIO>
cd Speecher
python -m venv .venv
.venv\Scripts\pip install faster-whisper nvidia-cublas-cu12 nvidia-cudnn-cu12
```

Depois ajuste `python` e `sidecar` no `%APPDATA%\Speecher\settings.json` para os caminhos do seu
clone, e escolha o provedor "Local" nas configurações. Na primeira execução o modelo (~1,6GB) é
baixado automaticamente.

## Arquitetura

```
[Hook de teclado global (Rust: WH_KEYBOARD_LL)]
        segura Ctrl+Win ──> [cpal grava WAV do microfone]
        solta ↓
[Transcrição (STT)]  Groq whisper-large-v3-turbo (nuvem, padrão)
                     ⇅ fallback automático bidirecional
                     faster-whisper large-v3-turbo local (GPU, sidecar Python residente)
        texto bruto ↓
[Reescrita]  Gemini Flash Lite (free tier, com fallback de modelo) + regras de edição
             + dicionário + perfil de escrita ativo
        texto final ↓
[Inserção]  clipboard + Ctrl+V sintético (com backup/restauração do clipboard)
```

Durante a gravação o overlay mostra as ondas; ao soltar o atalho ele passa a três pontos
("processando") e só some quando o texto entra. Se algo falhar no caminho, o motivo aparece
ali mesmo em vez de sumir no console. `Esc` durante a gravação descarta o ditado.

- **App**: Tauri 2 (Rust) + React/TypeScript — bandeja do sistema, overlay de ondas reativas
  ao microfone (2 estilos), janela de configurações com abas (Geral, Perfis,
  Dicionário, Snippets, Histórico, Diagnóstico), temas claro/escuro.
- **Sidecar STT local**: `sidecar/stt_server.py` — servidor HTTP local com faster-whisper na
  GPU; morre junto com o app (vigia o PID pai) e recusa porta duplicada.
- **Dados**: tudo local em `%APPDATA%\Speecher\` — settings.json com as chaves, history.jsonl.
  Histórico é desligável e apagável pela UI. As chaves são cifradas por usuário (DPAPI,
  prefixo `enc:`).
- **Plataforma**: camada específica isolada em `app/src-tauri/src/platform/windows.rs`
  (hook de teclado, proteção de chaves, atalho de colar, sidecar).

Já entregues desde a v0.1.0: instalador NSIS, autostart com toggle, idiomas de fala/saída com
tradução, dicionário com campos separados, single-instance, chaves criptografadas via DPAPI.
