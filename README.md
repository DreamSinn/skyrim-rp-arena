# Crônica de Sangue · Skyrim RP

Painel de torneio e apostas com chave dinâmica, visão pública, login administrativo e persistência no servidor.

## Executar localmente

```bash
node server.js
```

Abra `http://localhost:8770` para a visão pública.

O painel administrativo fica separado em `http://localhost:8770/admin`. A rota pública não exibe o botão nem o login administrativo. Mesmo que alguém descubra `/admin`, o servidor apresenta o login e mantém todas as alterações protegidas por sessão.

No primeiro acesso, a aplicação abre o cadastro do primeiro administrador. A senha deve ter pelo menos 8 caracteres. As credenciais não são gravadas no HTML: a senha é armazenada no servidor usando `scrypt` com salt.

## Recursos adicionados

- Chaves de **8, 16 ou 32 lutadores**.
- Geração automática de rodadas, confrontos e avanço dos vencedores.
- Atualização automática da visão pública a cada 5 segundos.
- Login administrativo com cookie de sessão HTTP-only.
- Alterações de apostas, regras, escalação e resultados protegidas por autenticação.
- Estado compartilhado em `data/state.json` no servidor.
- Administradores armazenados em `data/admins.json` com senha em hash.
- Visão pública aberta para os apostadores sem exigir login.
- Rotas separadas: `/` para espectadores e `/admin` para administradores.
- `robots.txt` bloqueia a indexação das rotas administrativas e da API.
- Toda ação administrativa exige sessão válida no frontend e no backend; sem login, cliques são bloqueados e gravações diretas na API retornam `401`.
- Na rota `/admin`, cancelar ou fechar o login mantém uma tela bloqueada; a visão pública e os controles administrativos não ficam acessíveis sem autenticação.
- Fluxo de transmissão em três passos: **embaralhar**, **sortear a luta** e **declarar o vencedor**.
- A visão pública exibe somente a luta atualmente selecionada, sem despejar a chave inteira na tela.
- Efeito visual e sinal sonoro gerado no navegador para vitórias e mudança de rodada.

## Protocolo de uma transmissão

1. No painel administrativo, clique em **Embaralhar agora** antes de qualquer aposta ou resultado.
2. Clique em **Sortear próxima luta**. Essa é a única luta liberada e visível para o público.
3. Registre apostas, se necessário, e clique em **Marcar vencedor**.
4. O resultado aparece para os espectadores com destaque visual e som; a próxima luta fica preparada.
5. Clique novamente em **Sortear próxima luta** para colocá-la no ar. Ao mudar de rodada, a arena mostra uma transição própria.

O quadro completo continua disponível no painel do administrador para conferência, mas não é exibido na transmissão pública.

## Resetar administradores

Para remover todos os acessos administrativos e permitir o cadastro de um novo administrador, faça um backup e apague somente `data/admins.json`:

```bash
cp data/admins.json data/admins.backup.json
rm data/admins.json
node server.js
```

Se a aplicação estiver rodando como serviço:

```bash
sudo systemctl stop cronica-arena
rm /home/ubuntu/skyrim-rp-arena/data/admins.json
sudo systemctl start cronica-arena
```

Depois, acesse `/admin` e crie o primeiro administrador novamente. Não apague `data/state.json`: esse arquivo contém a escalação, apostas, resultados e estado persistente do torneio.

## Divisão das apostas

- **Luta individual:** 75% para os apostadores vencedores, 15% para a organização e 10% para a bolsa dos lutadores.
- **Grupo / lado da chave:** 80% para os apostadores vencedores e 20% para a organização.
- **Campeão geral:** 85% para os apostadores vencedores e 15% para a organização.

Cada modalidade usa seu próprio pote e os valores são calculados no encerramento correspondente.

## Persistência entre computadores

O navegador não usa mais `localStorage` como fonte principal quando o servidor está disponível. O `localStorage` é apenas um cache de contingência. O estado oficial fica no servidor, portanto todos os computadores que acessarem a mesma URL verão os mesmos dados.

Para uso real no Discord, hospede o projeto em um servidor persistente com volume/disco permanente. O endereço temporário do Sandbox serve para demonstração durante a sessão e não deve ser tratado como URL de produção permanente.

## Arquivos

- `index.html`: interface completa e lógica do navegador.
- `server.js`: servidor HTTP, API, sessões e persistência.
- `data/state.json`: estado do torneio compartilhado.
- `data/admins.json`: administradores e hashes das senhas.
