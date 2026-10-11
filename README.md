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

## Modelo de odds e banca

As apostas de luta agora usam **odds decimais dinâmicas**, calculadas conforme o dinheiro colocado em cada lutador. A odd é registrada no instante da aposta e não muda depois. O painel exibe o retorno total e o lucro estimado:

- `retorno total = valor apostado × odd`
- `lucro = retorno total − valor apostado`
- A odd mínima é `1,10×` e a máxima é `20×`.
- A banca da casa é configurada no campo **Banca atual da casa**.
- Antes de aceitar uma aposta, o sistema calcula a maior exposição possível e recusa o registro se a banca não puder cobri-la.
- Ao encerrar uma luta, a banca é atualizada pelo resultado líquido: arrecadação menos pagamentos e bolsa dos lutadores.
- O painel **Apostas separadas por luta** mostra cada apostador, lutador escolhido, valor, odd travada e pagamento possível.


## Banca da casa e reserva (regra atual)

- Banca: 2.300 PO · Reserva mínima intocável: 1.000 PO · Banca operacional: 1.300 PO.
- Uma aposta só é recusada se o **pior resultado possível da luta** exigir mais do que a banca operacional (a reserva nunca é usada).
- Não existe mais a exigência de "lucro garantido nos dois lados": ela travava o mercado após 2 apostas. O pior resultado da casa aparece no painel e é limitado pela reserva.
- O encerramento da luta continua bloqueado se a banca final ficasse abaixo da reserva (rede de segurança).

## Lucro garantido da casa (regra atual)

- Em toda luta, o total pago aos vencedores é limitado a `Luta · vencedores`% do pool (padrão 85%); a bolsa fica com `Luta · bolsa`% (10%). A casa sempre fica com pelo menos `Luta · casa`% do pool (padrão 5%).
- As odds são fixadas na aposta e calculadas com folga abaixo desse teto (odd de luta equilibrada ≈ 1,50). Se a luta ficar muito desbalanceada e a soma prometida passar do teto, todos os vencedores são pagos proporcionalmente (pagamento em PO inteiros, arredondado para baixo).
- Com isso a casa não perde mais em nenhuma luta e a banca só sobe. A reserva de 1.000 PO fica como segurança extra.
- Os percentuais podem ser ajustados no painel (Luta · vencedores / casa / bolsa devem somar 100%).

## Transparência pública e regras (atualização)

- A tela pública agora mostra: apostas por luta (com a odd travada de cada apostador), **apostas nas chaves** (quem apostou em cada lado) e **palpites no campeão da grande final**. O admin tem os mesmos quadros.
- O carimbo "atualizado há Xs" no topo da arena mostra se a tela está recebendo dados; se cair a conexão ele avisa.
- As odds da luta atual só mudam com apostas **de luta** feitas naquela luta. Apostas de chave e de campeão não mexem nas odds. Sem apostas na luta, as odds ficam na abertura (≈ 1,50×) e a tela avisa isso.
- "Regras vigentes" é montado a partir das configurações salvas, então acompanha qualquer mudança feita no painel.
- Encerramento do torneio: pagamentos em PO inteiros; a sobra e os pagamentos não reclamados ficam com a casa e a **banca da casa é atualizada**. A casa precisa de pelo menos 1% em luta, chave e campeão (validado no painel e no servidor).

## Odds fixas por luta (opcional)

- No painel admin, seção **Odds fixas por luta**: para cada luta com os dois lutadores definidos dá para ligar/desligar odds fixas, definir a odd de cada lado e um limite de ouro por lutador. O padrão continua sendo odds dinâmicas com teto de pagamento (casa sempre ganha).
- Em luta com odd fixa o retorno mostrado é pago por inteiro (sem corte). A casa só aceita apostas que a banca operacional (banca − reserva) cobre no pior resultado, e recusa apostas acima do limite por lutador.
- Margem mínima: `1/oddA + 1/oddB` precisa ser pelo menos `1 / (vencedores%)` (≈ 1,176 com 85%), senão dá para apostar nos dois lados e lucrar. Painel e servidor recusam odds fora disso.
- O modo só muda antes da primeira aposta da luta (painel e servidor bloqueiam depois), para não alterar o que já foi prometido.
- Atenção: em odd fixa a casa **pode perder uma luta** se o lado mais apostado vencer. O limite por lutador e a reserva de 1.000 PO limitam o prejuízo.
