<div align="center">
<sub>🇬🇧 <a href="./README.md">English</a> &nbsp;|&nbsp; 🇨🇳 <a href="./README.zh-CN.md">中文</a> &nbsp;|&nbsp; 🇪🇸 <a href="./README.es.md">Español</a> &nbsp;|&nbsp; 🇯🇵 <a href="./README.ja.md">日本語</a> &nbsp;|&nbsp; 🇧🇷 <a href="./README.pt.md">Português</a></sub>
</div>

<br/>

<div align="center">

<img src="logo.png" alt="Perch" width="120">

# Perch

**Um gerenciador de processos e serviços para Windows, leve e de binário único, com web UI integrada.**

Escrito em Rust. Sem dependências de runtime. Um binário, duplo clique e pronto.

</div>

---

## Capturas de tela

### Menu interativo (duplo clique em `perch.exe`)

![menu](docs/screenshots/menu.png)

### Login

![login](docs/screenshots/login.png)

### Painel de processos

![processes](docs/screenshots/processes.png)

### Detalhes do processo — Transmissão de logs ao vivo

![process-detail](docs/screenshots/process-detail.png)

### Tarefas agendadas

![tasks](docs/screenshots/tasks.png)

---

## Por que Perch?

Gerenciar processos em segundo plano e serviços de longa duração no Windows é mais difícil do que deveria ser:

- **Task Scheduler** encerra por padrão processos de longa duração (timeouts de sessão, encerramento da árvore de processos, dependências de sessão de login) — nunca foi projetado para daemons.
- **NSSM / WinSw / Shawl** são um wrapper por serviço. Sem painel, sem visão central, registrar N vezes.
- **PM2** precisa de um runtime de Node.js, seu daemon inativo consome 40–80 MB, e é um cidadão de segunda classe no Windows (inicialização não confiável, gambiarras no registro).
- **Supervisor** nem funciona no Windows.

Perch preenche essa lacuna: **registrar uma vez, gerenciar tudo.** Um único serviço nativo supervisiona todos os seus processos, expõe uma web UI e fica fora do caminho.

## Recursos

- **Binário único** — sem runtime, sem dependências. A web UI é incorporada diretamente no executável.
- **Serviço nativo do Windows** — registra uma vez, inicia com o sistema, sobrevive ao logout, executa em Session 0.
- **Baixo consumo** — compilado em Rust, um único processo supervisor. Fica em torno de 4–5 MB de memória residente em repouso, contra os 40–80 MB do PM2.
- **Web UI integrada** — veja o status, inicie/pare/reinicie processos e acompanhe os logs pelo navegador. Gerencie remotamente, não apenas pelo console.
- **Reinício automático** — processos que falham são reiniciados automaticamente, com backoff configurável.
- **Tarefas agendadas** — agendamento estilo cron para trabalhos periódicos.
- **Rotação diária de logs** — logs da aplicação são escritos em `logs/perch.YYYY-MM-DD.log` ao lado do executável, formato delimitado por pipes.
- **Menu interativo** — duplo clique no exe para instalar/desinstalar/iniciar/parar o serviço e abrir a web UI, com elevação UAC sob demanda.

## Status

Perch já foi lançado e está pronto para uso, atualmente na versão v0.1.0. Por ser um projeto na fase 0.x, as interfaces e a configuração ainda podem mudar entre versões.

## Início rápido

Duplo clique em `perch.exe` e selecione `5) Open Web UI (Abrir Web UI)` no menu — Perch gerará automaticamente um `perch.toml` padrão (com um processo `demo` e o token `change-me`) e iniciará o servidor.

Ou pela linha de comando:

```
# Executar em primeiro plano (Ctrl-C para parar)
perch run

# Registrar Perch como serviço do Windows (precisa de admin)
perch install
perch start

# Abrir a web UI
http://127.0.0.1:9000
```

O token padrão é `change-me` — altere-o em `perch.toml` antes de qualquer uso real.

### Primeira execução: SmartScreen do Windows

O binário distribuído não é assinado digitalmente, então no primeiro uso o SmartScreen do Windows pode exibir **"O Windows protegeu o computador"**. Isso é esperado para ferramentas de código aberto não assinadas. Clique em **Mais informações → Executar assim mesmo** para continuar. (Baixar o `.zip` evita o aviso de download do navegador; o aviso do SmartScreen na primeira execução é outra etapa, inevitável sem um certificado de assinatura de código pago.)

### Executando o serviço com sua própria conta de usuário

Por padrão o serviço do Perch é executado como `LocalSystem`, que só vê variáveis de ambiente de nível de sistema, não o seu PATH de usuário (`HKCU\Environment`, por exemplo `%USERPROFILE%\.local\bin`, onde ficam ferramentas como `uv` ou CLIs instaladas via `pipx`). Se um processo gerenciado ou uma tarefa agendada chama um comando que depende de variáveis de ambiente de usuário, ele vai falhar com "command not found" sob o serviço, mesmo que funcione normalmente quando executado por você.

Para fazer o serviço rodar com sua própria conta em vez de `LocalSystem`:

1. Pressione `Win+R`, digite `services.msc` e pressione Enter.
2. Encontre **Perch Process Manager**, clique com o botão direito e selecione **Propriedades**.
3. Vá até a guia **Logon**.
4. Selecione **Esta conta**, informe sua conta do Windows (`.\seunome` para uma conta local, ou `DOMINIO\seunome` para uma de domínio) e sua senha (duas vezes).
5. Clique em OK; o Windows concede automaticamente à conta o direito "Efetuar logon como um serviço". Reinicie o serviço (botão direito → Reiniciar, ou `perch stop` seguido de `perch start`).

O serviço agora roda como você, então os processos filhos herdam todo o seu ambiente de usuário, incluindo o PATH de usuário.

Ressalvas:
- Se você alterar sua senha do Windows depois, o serviço vai falhar ao iniciar até que você atualize a senha na mesma guia Logon.
- Alternativamente, evite a troca de conta usando caminhos absolutos para os executáveis nos comandos do processo/tarefa, ou definindo explicitamente as variáveis necessárias.

## Segurança

O endpoint de gerenciamento pode iniciar e parar processos arbitrários — ele tem **privilégios muito altos**. Perch vincula-se a `127.0.0.1` por padrão e exige um token de acesso. Não exponha a porta de gerenciamento a redes não confiáveis.

## Comparação

| | Perch | NSSM / WinSw | PM2 | Supervisor |
|---|:---:|:---:|:---:|:---:|
| Binário único | ✅ | ✅ | ❌ (precisa de Node) | ❌ (precisa de Python) |
| Gerencia muitos processos de uma instância | ✅ | ❌ (1 por serviço) | ✅ | ✅ |
| Web UI integrada | ✅ | ❌ | ✅ | ✅ |
| Serviço nativo do Windows | ✅ | ✅ | ⚠️ | ❌ |
| Tarefas agendadas | ✅ | ❌ | ✅ | ⚠️ |
| Rotação diária de logs | ✅ | ❌ | ⚠️ | ⚠️ |
| Baixo consumo de memória | ✅ | ✅ | ❌ | ⚠️ |

## Compilar a partir do código-fonte

```
cargo build --release
```

Requer uma toolchain estável recente de Rust.

## Licença

[MIT](LICENSE) — veja o arquivo [LICENSE](LICENSE) para mais detalhes.
