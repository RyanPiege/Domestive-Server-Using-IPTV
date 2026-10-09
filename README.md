# Servidor SFTP de Baixo Custo com Android TV

Um **servidor de arquivos próprio e permanente**, montado com uma TV Box/Android TV que já existe em casa, um HD externo USB e software 100% gratuito (Termux + OpenSSH). Sem mensalidade de nuvem, sem servidor dedicado, sem hardware novo.

> Base para um servidor de mídia e IPTV doméstico.

---

## O problema

Guardar, compartilhar e acessar arquivos em rede costuma exigir uma de duas coisas: **assinatura mensal de nuvem** ou **hardware dedicado** (NAS, mini PC, servidor). Para uso doméstico ou de pequena escala, ambos são mais caros do que o necessário.

## A solução

Reaproveitar o que já está ligado 24 horas na sala:

```
Notebook / PC
     │  SFTP · porta 8022 · rede local
     ▼
Android TV / TV Box  (Termux + OpenSSH)
     │
     ▼
HD externo USB  →  /storage/<VOLUME>/CLOUD
```

| Componente | Origem | Custo |
|------------|--------|-------|
| Android TV / TV Box | Já existente | R$ 0 |
| HD externo USB | Já existente | R$ 0 |
| Termux + OpenSSH | Software livre | R$ 0 |
| WinSCP (cliente gráfico, opcional) | Software livre | R$ 0 |
| Termux:Boot (inicialização automática) | Software livre | R$ 0 |

> Se algum item precisar ser comprado (HD, TV Box), o investimento é único, sem recorrência.

## Comparativo de custo

| | Nuvem por assinatura | NAS / mini PC dedicado | **Este projeto** |
|---|---|---|---|
| Investimento inicial | Nenhum | [R$ __] | **[R$ __ ou R$ 0 se reaproveitar]** |
| Custo recorrente | Mensalidade | Energia | **Energia (baixo consumo)** |
| Capacidade | Limitada ao plano | Depende dos discos | **Tamanho do HD** |
| Controle dos dados | Terceiros | Próprio | **Próprio** |
| Funciona sem internet | Não | Sim | **Sim (rede local)** |

> Preencha os valores `[R$ __]` com os preços da sua região antes de apresentar.

## Vantagens

- **Custo próximo de zero** quando se reaproveita hardware
- **Dados sob seu controle**, sem depender de plano ou provedor
- **Baixo consumo de energia**, próprio de TV Box
- **Acesso por padrões abertos** (SFTP): funciona com PowerShell, WinSCP e qualquer cliente SFTP
- **Inicia sozinho** ao ligar a TV Box (Termux:Boot)
- **Escalável**: trocar ou ampliar o HD amplia a capacidade

## Limitações (transparência)

- Não substitui um NAS profissional em desempenho, redundância (RAID) ou suporte
- A TV Box pode matar processos em segundo plano: exige ajuste de bateria/otimização
- A escrita no HD depende das permissões do Android para o volume USB
- O IP pode mudar por DHCP: resolvido com **reserva DHCP** no roteador
- Isolamento por usuário (RYAN / YCE) ainda é etapa futura

## Como funciona (resumo)

1. Instalar o Termux e o OpenSSH: `pkg install openssh`
2. Definir a senha e o usuário: `passwd` e `whoami`
3. Liberar o armazenamento: `termux-setup-storage`
4. Iniciar o servidor: `sshd` (porta padrão **8022**)
5. Conectar de outro computador: `sftp -P 8022 USUARIO@IP_DA_TV`
6. Automatizar com Termux:Boot e reservar o IP no roteador

Passo a passo completo, testes e troubleshooting: **[MANUAL_TECNICO.md](MANUAL_TECNICO.md)**.

## Diagnóstico rápido

```
Rede:          Test-NetConnection IP -Port 8022
Serviço:       pgrep -a sshd
Armazenamento: touch /storage/<VOLUME>/CLOUD/RYAN/teste.txt
```

## Segurança

- Não expor a porta 8022 diretamente à internet
- Usar senha forte e exclusiva
- Manter Termux e OpenSSH atualizados
- Encerrar o servidor quando não for necessário: `pkill sshd`

## Licença

Defina a licença do projeto (ex.: MIT).
