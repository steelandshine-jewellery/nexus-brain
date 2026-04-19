# Protocolo de Handover Multi-PC (Antigravity)

**Contexto**: O ecossistema desenvolve-se através de múltiplos computadores físicos, mantendo interações isoladas com a inteligência artificial (Antigravity). O `core-server` (192.168.1.131) é a única Fonte de Verdade central (Source of Truth) a nível de código base e infraestrutura, mas a memória de conversação ("curto prazo") e os ficheiros sintéticos do Antigravity são restritos fisicamente a cada PC.

## O Desafio da Amnésia Local
Se uma tarefa sistémica for iniciada no PC A e o utilizador migrar abruptamente para o PC B, a instância Antigravity dessa segunda máquina não tem histórico contextual imediato do chat, nem acesso aos *Knowledge Items* (KI) acabados de iterar na primeira máquina. 

## A Regra de Ouro da Transição Perfeita
Para garantir a continuidade intocável do raciocínio durante a mudança entre estações de trabalho, deve-se cumprir este protocolo:

1. **No PC de encerramento (PC A):** 
   - Antes de fechar a sessão com o Antigravity, ordenar de forma expressa: *"Faz um resumo do nosso progresso da sessão de hoje e regista-o num ficheiro no cofre do Obsidian"*.
   - Eu (Antigravity) compilarei a memória de curto prazo e assegurarei que ela é cristalizada nalgum documento do cofre (por exemplo, criando um log de sessão) e enviada por Git para a nuvem.

2. **No PC de início (PC B):** 
   - Ao arrancar uma nova sessão, garante que o repositório `nexus-brain` local sincronizou os novos commits (pull automático ou manual).
   - Inicia o chat com o novo contexto: *"Lê o report de atividade recente guardado no Obsidian e reassume o plano a partir de onde ficámos"*.

## Consequência 
- O estado lógico não se perde na arquitetura local.
- O Antigravity da estação secundária carrega o histórico para a sua nova "memória de curto prazo" logo na primeira iteração, evitando redundâncias, pesquisas e reconstrução da mesma fase de pensamento.
