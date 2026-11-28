---
title: "Storage ZFS em hardware reciclado: testar e recuperar discos antigos no TrueNAS SCALE (Parte 2)"
date: 2026-11-11
draft: true
summary: "TrueNAS, ZFS em mirror e uma semana a descobrir que três discos 'PASSED' não eram de confiança — com todos os comandos."
tags: ["homelab", "truenas", "zfs", "smart", "storage", "nfs"]
featureImage: "img/capa-post-02.png"
series: ["Homelab a Custo Zero"]
showTableOfContents: true

---

*Post 2 da série. No [post 1]({{< ref "post-01-inventario-e-arquitetura" >}}) apresentei o inventário e as decisões de arquitetura. Aqui conto a fase do storage, com os erros incluídos — e foram vários.*

O plano parecia simples: instalar o TrueNAS num PC antigo, juntar os discos num pool ZFS e partilhar o espaço com o Proxmox por NFS. A parte do software correu bem. A parte dos discos deu muito mais que contar.

## O ponto de partida

Tinha quatro discos de 1TB para os dados, todos com muitas horas de uso, e um SSD pequeno para o sistema. Antes de lhes confiar fosse o que fosse, corri o `smartctl` em cada um:

```
sudo smartctl -a /dev/sdX
```

Os três primeiros resultados:

| Disco | Horas de uso | Setores realocados | Setores pendentes | Uncorrectable | Resultado SMART |
|---|---|---|---|---|---|
| Seagate ST31000528AS | 69 823 (~8 anos) | **434** | **131** | **131** | PASSED |
| WD Red WD10EFRX | 66 613 (~7,6 anos) | 0 | **1** | 0 | PASSED |
| WD Red WD10EFRX | 34 400 (~3,9 anos) | 0 | 0 | 0 | PASSED |

Os três dizem `PASSED`, e é aqui que está a primeira lição: **o `PASSED` só significa que nenhum atributo cruzou o limiar do fabricante**, que é muito permissivo. O Seagate estava claramente a falhar, com centenas de setores realocados e mais de cem pendentes. O que interessa são os contadores `Reallocated_Sector_Ct`, `Current_Pending_Sector` e `Offline_Uncorrectable`, que devem estar a zero.

> **Erro meu:** na primeira ronda, copiei o output do *mesmo* disco três vezes. O que me denunciou foi o número de série, igual nos três ficheiros. Confirma sempre o `Serial Number` no topo de cada output.

## O teste longo falhou nos três

O `smartctl -a` mostra os contadores, mas só um teste longo lê o disco todo:

```
sudo smartctl -t long /dev/sdX
sudo smartctl -l selftest /dev/sdX
```

Demora horas (cerca de 2,5 a 3 por disco). Resultado: **os três falharam**, com `Completed: read failure` e o primeiro setor com erro (o LBA) indicado em cada um. Incluindo o WD de 3,9 anos que parecia limpo.

Um pormenor curioso do log: o `LifeTime(hours)` mostrava 4288 e 1077 em vez de 69 823 e 66 613. É o contador de 16 bits a dar a volta aos 65 536 horas. Não é um erro novo, é só truncagem.

## Disco avariado ou problema à volta do disco?

Três discos de marcas e idades diferentes a falharem ao mesmo tempo cheira a causa comum: fonte, cabos ou controlador. Em vez de adivinhar, investiguei:

1. **Os contadores não mudaram** depois dos testes.
2. **O `dmesg` estava limpo:** nem `hard resetting link`, nem `failed command`, nem `SError`. Cabos e alimentação pareciam inocentes.
3. **Ler diretamente o setor que falhou**, com o LBA que o teste indicou:

```
sudo dd if=/dev/sdc of=/dev/null bs=512 skip=<LBA> count=1 iflag=direct
```

Deu `Input/output error`, depois de quase 3 segundos de tentativas. O `dmesg` passou a mostrar `Unrecovered read error - auto reallocate failed`, e o `Current_Pending_Sector` do disco subiu para 1. **O problema era mesmo da superfície do disco.**

Um aviso: usei o LBA de um disco no outro numa das tentativas, e o `dd` "passou". Um teste com o endereço errado não prova nada.

## Reescrever os setores fracos

Um setor pendente é um setor que o disco não consegue ler, mas que só decide realocar quando alguém **escreve** lá. Como os discos não tinham dados que me importassem, escrevi-os por inteiro, o que obriga o disco a remapear qualquer setor mau, e não só os que eu conhecia:

```
sudo dd if=/dev/zero of=/dev/sdX bs=1M oflag=direct status=progress
```

⚠️ **Isto apaga tudo.** Confirma o disco pelo número de série antes de carregares no Enter.

Demorou cerca de 2h30 num disco e 2h10 no outro, a uns 110-130 MB/s. No fim, nos dois WD Red, o `Current_Pending_Sector` voltou a **0**, e o `Reallocated_Sector_Ct` ficou também a **0**. Eram setores "fracos" que, depois de reescritos, passaram a ser legíveis.

O Seagate, com centenas de setores realocados e pendentes, não teve essa sorte: é um disco em degradação ativa, e foi **descartado**. Ficou fora do pool, e nem como disco de reserva o quis.

## O disco de substituição

Para ficar com três discos, usei outro de 1TB que já tinha em casa (custo zero, como sempre): um WD Green WD10EACS, com cerca de 15 mil horas, limpo no SMART. Dois pormenores:

- O `Load_Cycle_Count` era de 3114, muito baixo. Os discos Green têm fama de estacionar as cabeças a toda a hora, e esse contador mostra se o disco já sofreu com isso.
- O **ERC estava desligado** (`smartctl -l scterc` mostrava `Disabled`). Sem esse limite, o disco pode ficar muito tempo a tentar reler um setor mau, e o sistema acaba por o tratar como avariado. Limitei a 7 segundos:

```
sudo smartctl -l scterc,70,70 /dev/disk/by-id/<disco>
```

Esta definição perde-se quando o disco desliga, por isso, se a quiseres permanente, põe o comando num script de arranque.

## Uma falha de energia e o controlador em modo IDE

A meio dos novos testes longos, **faltou a luz em casa**. Os testes foram abortados, e tive de os repetir. Aproveitei a paragem para corrigir uma coisa que o `dmesg` me mostrara: o controlador SATA estava em modo **IDE** (dois discos no mesmo canal, `NCQ depth 0/32`). Mudei-o para **AHCI** na BIOS, e o sistema passou a ver o controlador como `ahci`, com NCQ ativo.

Depois de repetir os testes longos, os **três discos passaram** com `Completed without error`, e os contadores ficaram todos a zero.

> **Lição de nomes:** depois de reiniciar, as letras `sda`, `sdb` e `sdc` trocaram de disco. Usa sempre o número de série ou o caminho em `/dev/disk/by-id/`.

## A decisão do pool: mirror ou RAIDZ1?

| | RAIDZ1 (3 discos) | Mirror (2 discos + 1 de reserva) |
|---|---|---|
| Capacidade útil | ~1,8 TiB | ~0,9 TiB |
| Tolera | 1 disco a falhar | 1 disco a falhar |
| Desempenho para VMs | IOPS aleatórios fracos | Melhor, sobretudo em leitura |
| Risco | Uma 2ª falha durante o resilver perde tudo | Menor, e o disco de reserva está à mão |

Escolhi o **mirror com os dois WD Red**, deixando o WD Green de reserva. O storage vai servir VMs, onde o desempenho em IOPS conta mais do que a capacidade, e com discos de idades tão diferentes prefiro um resilver simples. O preço é ficar com cerca de 0,9 TiB.

*(Screenshot do pool em Storage — [inserir imagem])*

## Datasets e partilhas NFS

Dentro do pool (criado no NAS gringotts) criei um dataset principal, `hogwarts` — o nome do servidor Proxmox que o vai usar — e, dentro dele, um dataset por finalidade, cada um com a sua partilha NFS:

- `dormitories`: discos das VMs
- `restricted-section`: backups
- `library`: ISOs e templates

Duas definições importam em cada partilha: os **hosts autorizados** (só o IP do Proxmox) e o **Maproot User/Group a `root`**, sem o qual o Proxmox, que escreve como root, apanha erros de permissões.

Um dataset por finalidade, em vez de uma única partilha, dá-me snapshots, quotas e permissões independentes. E um pormenor: o NFS não atravessa a fronteira entre datasets ZFS, por isso cada um precisa da sua partilha.

## Os testes SMART passaram a ser tarefas cron

Quis agendar testes curtos semanais e longos mensais, como se fazia sempre, mas no TrueNAS 25.10.7, que é a versão que tenho, **a interface de testes SMART já não existe**. Segundo as notas da versão 25.10, os testes agendados passaram a tarefas cron. Ficou assim, com a minha tarefa semanal (teste curto, aos domingos às 03:00):

```
for d in /dev/disk/by-id/ata-*; do case "$d" in *-part*) continue;; esac; smartctl -t short "$d"; done
```

e a mensal (teste longo, só nos discos WD, no dia 10 às 02:00):

```
for d in /dev/disk/by-id/ata-WDC_*; do case "$d" in *-part*) continue;; esac; smartctl -t long "$d"; done
```

Usei o caminho `by-id` por causa da troca de letras, e o `case` para ignorar as partições. As tarefas cron correm como `root`; quando testei os comandos à mão sem `sudo`, apanhei `Permission denied`.

Um aviso: estas tarefas só **lançam** os testes, e não me avisam se falharem. Por isso, olho para o log depois de cada teste longo (`smartctl -l selftest`).

## O que aprendi (e o que falta)

- Um `PASSED` não prova nada. Olha para os contadores e faz um teste longo.
- Confirma o disco pelo número de série antes de qualquer comando destrutivo.
- Verifica o modo do controlador (AHCI) e o ERC dos discos.
- Para reparar um disco que não tem dados, escrever o disco inteiro pode resolver setores fracos. Para dados importantes, copia primeiro.
- Uma UPS, ou pelo menos uma proteção de energia, faz falta: a falha de luz custou-me horas de testes.
- O NAS tem **só 4GB de RAM**, abaixo do recomendado para ZFS. Vai ser uma limitação para medir mais à frente.

No próximo post: a rede. Vou limpar a configuração antiga dos dois Catalyst 3750, ligá-los em stack e configurar as VLANs de gestão, storage e VMs. Depois é a vez do Proxmox, que vai usar este storage por NFS.

---

*Hardware usado: HP Compaq Pro 6300 MT (i3-3220, 4GB RAM), 2× WD Red WD10EFRX, 1× WD Green WD10EACS, SSD Kingston de 120GB para o sistema.*
