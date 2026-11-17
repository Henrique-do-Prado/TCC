# Performance e Benchmark do Sistema FUSE no Linux

Código do TCC de Henrique do Prado (Sistemas de Informação, IFRS Campus Vacaria): coleta e análise dos experimentos que comparam o FUSE *pass-through* (`passthrough_hp`) com o ext4 nativo.

## Ambiente usado

- Pop!_OS 22.04, kernel 7.1.1-76070101-generic (o kernel padrão da distribuição na época, instalado pelas atualizações normais do sistema)
- libfuse 3.10.5 e fio 3.28
- ext4 sobre LUKS + LVM (padrão de instalação do Pop!_OS)

## Dependências

```bash
sudo apt install fio sysstat nvme-cli fuse3 openssl tmux git meson ninja-build pkg-config g++ python3-matplotlib
```

## Compilar o passthrough_hp

O pacote `fuse3` da distribuição não traz o `passthrough_hp`, que fica nos exemplos do código-fonte da libfuse. Use a mesma versão (3.10.5):

```bash
git clone --depth 1 --branch fuse-3.10.5 https://github.com/libfuse/libfuse.git
cd libfuse
mkdir build && cd build
meson setup ..          # em versões antigas do meson: meson ..
ninja example/passthrough_hp
./example/passthrough_hp --help
```

O binário fica em `libfuse/build/example/passthrough_hp`. Passe esse caminho aos scripts pela variável `PT_HP`.

## Scripts

| Arquivo | O que faz |
|---|---|
| `coleta.sh` | Roda uma repetição completa da matriz (400 execuções do fio, cerca de 5,3 h) em ordem aleatória. Uma repetição por vez, com `REP=1` a `5`. Confere kernel e perfil de energia, grava o JSON do fio, o iostat e o SMART de cada execução e pode ser retomado se for interrompido. |
| `analisa_coleta.py` | Lê as repetições, confere a coleta, calcula medianas, razões e o teste de Mann-Whitney, classifica cada célula e gera `execucoes.csv`, `celulas.csv` e os heatmaps. |
| `figuras_tese.py` | Gera as figuras do Capítulo 4 a partir dos dois CSVs. |
| `piloto.sh`, `analisa_piloto.py` | Estudo piloto, usado para calibrar o warm-up e a duração das execuções. |
| `testes_p1_p4.sh`, `analisa_p1_p4.py` | Testes complementares: controle da leitura aleatória de 1 MiB (dica de `fadvise` e readahead) e contagem das consultas de atributos com strace. |

## Uso rápido

Antes de rodar, ajuste no início do `coleta.sh` os caminhos `SRC_DIR`, `MNT` e `DEV` e o `KERNEL_ESPERADO` (o script para se o kernel em uso for outro). Cada repetição escreve cerca de 1,3 TB no SSD.

```bash
# como root, em modo texto e na tomada
sudo systemctl isolate multi-user.target
system76-power profile performance

# teste rápido (10 execuções curtas)
sudo TESTE=1 PT_HP=/caminho/passthrough_hp ./coleta.sh

# uma repetição por sessão
sudo REP=1 PT_HP=/caminho/passthrough_hp \
     systemd-inhibit --what=sleep:idle:handle-lid-switch ./coleta.sh

# depois das 5 repetições
python3 analisa_coleta.py resultados_coleta
python3 figuras_tese.py resultados_coleta/analise figuras/
```
