**template-somadorpf-vhdl**

# Tutorial: Implementação de Somador Ponto Flutuante na DE10-Lite

**Autores:** `[Nome do Aluno 1]`, `[Nome do Aluno 2]`, `[Nome do Aluno 3]`
**Disciplina:** Sistemas Digitais Q2.20026
**Data:** `[Data da entrega]`

---

> **Arquivos do projeto**
>
> | Arquivo | Entidade | Papel |
> |---|---|---|
> | `final_fp_adder.vhd` | `fp_adder_v2` | Núcleo aritmético combinacional (soma em ponto flutuante) |
> | `hex_to_7seg.vhd` | `hex_to_7seg` | Decodificador de nibble para display de 7 segmentos |
> | `final_fp_adder_de10.vhd` | `fp_adder_de10` | Wrapper / top-level da DE10-Lite (SW, KEY, HEX) |
> | `tb_fp_adder.vhd` | `tb_fp_adder` | Testbench do núcleo |
> | `tb_fp_adder_de10.vhd` | `tb_fp_adder_de10` | Testbench de integração (protocolo SW/KEY + displays) |
> | `de10_lite_final.qsf` | — | Assignments de pinos e padrões de I/O da DE10-Lite |
> | `adder_wave.vcd` / `wave.vcd` | — | Formas de onda das simulações (núcleo / wrapper) |

---

*Etapa 1*

## 1. Objetivo do Projeto

Este projeto adapta o somador de ponto flutuante simplificado (13 bits) do livro-texto para a placa Terasic DE10-Lite (MAX 10). O objetivo é demonstrar a síntese lógica e a simulação de hardware usando VHDL.

Concretamente, o trabalho consistiu em:

1. **Validar** o algoritmo original em VHDL por simulação (GHDL + GTKWave), antes de tocar em qualquer coisa ligada a hardware;
2. **Corrigir** três comportamentos do núcleo original que se mostraram inadequados para os casos de teste escolhidos (zero não canônico, zero negativo e descarte de resultados menores que 0,5);
3. **Adaptar** o projeto para a DE10-Lite, criando um wrapper que resolve a limitação física de haver apenas 10 chaves para 26 bits de entrada, e um decodificador hexadecimal para os displays de 7 segmentos;
4. **Comprovar por simulação** que a adaptação de hardware não quebrou a lógica matemática, com um segundo testbench que estimula o circuito exatamente pelo protocolo físico (`SW`/`KEY`) e verifica os padrões dos displays;
5. **Documentar** todo o processo, incluindo o uso de IA.

O sistema final recebe dois operandos de 13 bits, produz a soma no mesmo formato e exibe sinal, expoente e significância em quatro displays de 7 segmentos.

---

## 2. Descrição gráfica do funcionamento do sistema

### 2.1 Formato numérico de 13 bits

```text
 12      11        8 7                       0
+---------+---------+-------------------------+
| S (1)   | E (4)   | F (8)                   |
| sinal   | expoente| significância explícita |
+---------+---------+-------------------------+
```

A convenção adotada — a mesma usada pelos comentários dos testbenches e pelas asserções de validação — é:

$$\text{valor} = (-1)^S \times \frac{F}{256} \times 2^{E}$$

onde `S` é o bit de sinal, `F` é o valor inteiro sem sinal dos 8 bits de `frac` e `E` é o valor inteiro sem sinal dos 4 bits de `exp`. Ou seja, os 8 bits de `frac` são lidos como `0.FFFFFFFF₂`. **Não há** bit implícito, bias de expoente, NaN, infinito ou arredondamento formal: o formato é próprio do projeto e não é IEEE 754.

| Campo | Largura | Faixa | Interpretação |
|---|---|---|---|
| `S` | 1 bit | 0 ou 1 | `0` = positivo, `1` = negativo |
| `E` | 4 bits | 0 a 15 | inteiro sem sinal, sem bias, sem valores negativos |
| `F` | 8 bits | 0 a 255 | `0.FFFFFFFF₂`, isto é, `F/256` |

**Exemplos de codificação canônica** (valores usados nos testes):

| Valor | `S` | `E` | `F` | Conferência | Displays `HEX3..HEX0` |
|---|---|---|---|---|---|
| 0,25 | 0 | 0000 | 01000000 | 64/256 × 2⁰ = 0,25 | `0040` |
| 0,5 | 0 | 0000 | 10000000 | 128/256 × 2⁰ = 0,5 | `0080` |
| 0,75 | 0 | 0000 | 11000000 | 192/256 × 2⁰ = 0,75 | `00C0` |
| 1,0 | 0 | 0001 | 10000000 | 0,5 × 2¹ = 1,0 | `0180` |
| 1,5 | 0 | 0001 | 11000000 | 0,75 × 2¹ = 1,5 | `01C0` |
| −1,5 | 1 | 0001 | 11000000 | −0,75 × 2¹ = −1,5 | `11C0` |
| 128 | 0 | 1000 | 10000000 | 0,5 × 2⁸ = 128 | `0880` |
| 16320 | 0 | 1110 | 11111111 | 255/256 × 2¹⁴ = 16320 | `0EFF` |
| 32640 (máximo) | 0 | 1111 | 11111111 | 255/256 × 2¹⁵ = 32640 | `0FFF` |
| 0,00390625 (mínimo ≠ 0) | 0 | 0000 | 00000001 | 1/256 × 2⁰ | `0001` |

> **Nota de convenção.** Uma parte do material interno do grupo descreve a significância como `Q1.7` (`F/128 × 2^E`, com o ponto binário depois de `frac(7)`). As duas convenções descrevem o *mesmo hardware* — o circuito não conhece a posição do ponto binário — e diferem apenas por um fator 2 na leitura do valor. Este relatório adota `F/256 × 2^E` porque é a convenção usada nos comentários e nos vetores de teste do código final (por exemplo, `0 | 1110 | 11111111` é documentado como 16320 no `tb_fp_adder_de10.vhd`).

### 2.2 Diagrama de blocos do sistema completo

```mermaid
flowchart TD
    SW["SW9..SW0<br/>(modo, reset, dados)"] --> REG1["Registrador operando 1<br/>sign1 / exp1 / frac1"]
    SW --> REG2["Registrador operando 2<br/>sign2 / exp2 / frac2"]
    KEY0["KEY0 (borda de subida)"] --> REG1
    KEY1["KEY1 (borda de subida)"] --> REG2
    REG1 --> ADD["fp_adder_v2<br/>(combinacional)"]
    REG2 --> ADD
    ADD --> MUX["Seleção de exibição<br/>reset > preview1 > preview2 > resultado"]
    SW --> MUX
    KEY0 --> MUX
    KEY1 --> MUX
    MUX --> DEC["4 × hex_to_7seg"]
    DEC --> HEX["HEX3=S | HEX2=E | HEX1=F(7:4) | HEX0=F(3:0)"]
```

### 2.3 Datapath interno do núcleo `fp_adder_v2`

O núcleo é **puramente combinacional**: não há clock nem registradores entre as etapas. As "4 etapas" são estágios lógicos, não estágios de pipeline.

```mermaid
flowchart TD
    IN["sign1, exp1, frac1<br/>sign2, exp2, frac2"] --> S1["Etapa 1 — Ordenação por magnitude<br/>(exp1 &amp; frac1) &gt; (exp2 &amp; frac2)<br/>→ big_* / small_*"]
    S1 --> S2["Etapa 2 — Alinhamento<br/>diff_exp = big_exp − small_exp<br/>align_frac = small_frac &gt;&gt; diff_exp"]
    S2 --> S3["Etapa 3 — Soma ou subtração<br/>frac_sum(8:0) = big_frac ± align_frac"]
    S3 --> S4A["Etapa 4a — Contagem de zeros à esquerda<br/>leado (codificador de prioridade)"]
    S4A --> S4B["Etapa 4b — normal_sum = frac_sum &lt;&lt; leado"]
    S4B --> S4C["Etapa 4c — Processo de normalização<br/>(4 casos)"]
    S3 --> S4C
    S4C --> OUT["sign_out, exp_out, frac_out"]
```

**Sinais internos relevantes** (todos declarados em `fp_adder_v2`):

| Sinal | Largura | Função |
|---|---|---|
| `big_sign` / `small_sign` | 1 bit | sinais do operando de maior / menor magnitude |
| `big_exp` / `small_exp` | 4 bits | expoentes ordenados por magnitude |
| `big_frac` / `small_frac` | 8 bits | significâncias ordenadas por magnitude |
| `diff_exp` | 4 bits | `big_exp - small_exp`, quantidade de deslocamentos do alinhamento |
| `align_frac` | 8 bits | significância menor já deslocada à direita |
| `frac_sum` | **9 bits** | resultado da soma/subtração; o bit extra guarda o carry |
| `leado` | 3 bits | número de zeros à esquerda de `frac_sum(7 downto 0)` |
| `normal_sum` | 8 bits | `frac_sum` deslocada à esquerda por `leado` |
| `normal_exp` / `normal_frac` | 4 / 8 bits | expoente e significância já normalizados |

### 2.4 Tabela de decisão — Etapa 1 (ordenação)

A comparação usa apenas `exp & frac`; o bit de sinal **não** participa, porque a etapa procura a maior **magnitude**, não o maior valor algébrico.

| Condição | `big_*` recebe | `small_*` recebe |
|---|---|---|
| `(exp1 & frac1) > (exp2 & frac2)` | operando 1 | operando 2 |
| caso contrário (inclui empate) | operando 2 | operando 1 |

Consequência: em um empate exato de magnitude com sinais opostos, `big_sign` pode ser `1`. É por isso que o zero canônico precisa ser forçado na saída (ver §3.1).

### 2.5 Tabela verdade — Etapa 2 (alinhamento)

| `diff_exp` | `align_frac` | Efeito |
|---|---|---|
| `0000` | `small_frac` | sem deslocamento |
| `0001` | `0 & small_frac(7:1)` | `>> 1` |
| `0010` | `00 & small_frac(7:2)` | `>> 2` |
| `0011` | `000 & small_frac(7:3)` | `>> 3` |
| `0100` | `0000 & small_frac(7:4)` | `>> 4` |
| `0101` | `00000 & small_frac(7:5)` | `>> 5` |
| `0110` | `000000 & small_frac(7:6)` | `>> 6` |
| `0111` | `0000000 & small_frac(7)` | `>> 7` |
| `others` (≥ 8) | `00000000` | menor operando descartado |

### 2.6 Tabela verdade — Etapa 3 (soma/subtração)

| `big_sign` vs `small_sign` | `frac_sum` (9 bits) |
|---|---|
| iguais | `('0' & big_frac) + ('0' & align_frac)` |
| diferentes | `('0' & big_frac) - ('0' & align_frac)` |

Como a Etapa 1 já garantiu `big_frac ≥ align_frac` em magnitude, a subtração nunca precisa representar um valor negativo.

### 2.7 Tabela verdade — Etapa 4a (`leado`, codificador de prioridade)

| Primeiro `1` em | `leado` | Deslocamentos à esquerda necessários |
|---|---|---|
| `frac_sum(7)` | `000` | 0 |
| `frac_sum(6)` | `001` | 1 |
| `frac_sum(5)` | `010` | 2 |
| `frac_sum(4)` | `011` | 3 |
| `frac_sum(3)` | `100` | 4 |
| `frac_sum(2)` | `101` | 5 |
| `frac_sum(1)` | `110` | 6 |
| nenhum dos acima | `111` | 7 |

### 2.8 Tabela de decisão — Etapa 4c (normalização, 4 casos)

Esta é a tabela mais importante do projeto: é aqui que estão as correções em relação ao código original.

| # | Condição | `normal_exp` | `normal_frac` | Situação física |
|---|---|---|---|---|
| **1** | `frac_sum = "000000000"` | `0000` | `00000000` | Cancelamento exato → zero canônico |
| **2** | `frac_sum(8) = '1'` | `big_exp + 1` | `frac_sum(8 downto 1)` | Carry na soma → desloca 1 à direita |
| **3** | `leado > big_exp` | `0000` | `shift_left(frac_sum(7:0), big_exp)` | Normalização completa exigiria expoente negativo → normalização **parcial**, para em `E = 0` |
| **4** | caso contrário | `big_exp - leado` | `normal_sum` | Normalização convencional (`0.1xxxxxxx`) |

E a saída:

| Condição | `sign_out` |
|---|---|
| `normal_frac = "00000000"` | `'0'` (zero sempre positivo) |
| caso contrário | `big_sign` |

### 2.9 Região não normalizada em `E = 0`

Como consequência direta do **Caso 3**, quando o expoente chega a zero a normalização é interrompida e a significância é preservada mesmo com `frac(7) = 0`. Isso cria uma região de valores pequenos (conceitualmente análoga à ideia de subnormais, sem ser a implementação formal do IEEE 754):

| Codificação | Valor |
|---|---|
| `0 \| 0000 \| 10000000` | 0,5 |
| `0 \| 0000 \| 01000000` | 0,25 |
| `0 \| 0000 \| 00100000` | 0,125 |
| `0 \| 0000 \| 00010000` | 0,0625 |
| `0 \| 0000 \| 00001000` | 0,03125 |
| `0 \| 0000 \| 00000100` | 0,015625 |
| `0 \| 0000 \| 00000010` | 0,0078125 |
| `0 \| 0000 \| 00000001` | 0,00390625 (menor valor não nulo) |

### 2.10 Tabela verdade do `hex_to_7seg`

Saída de 8 bits `segments(7 downto 0) = dp g f e d c b a`, **ativa em nível baixo** (`0` acende o segmento, `1` apaga). O bit 7 (ponto decimal) fica sempre desligado.

| `value` | `segments` | Dígito | `value` | `segments` | Dígito |
|---|---|---|---|---|---|
| `0000` | `11000000` | 0 | `1000` | `10000000` | 8 |
| `0001` | `11111001` | 1 | `1001` | `10010000` | 9 |
| `0010` | `10100100` | 2 | `1010` | `10001000` | A |
| `0011` | `10110000` | 3 | `1011` | `10000011` | b |
| `0100` | `10011001` | 4 | `1100` | `11000110` | C |
| `0101` | `10010010` | 5 | `1101` | `10100001` | d |
| `0110` | `10000010` | 6 | `1110` | `10000110` | E |
| `0111` | `11111000` | 7 | `1111` | `10001110` | F |
| — | `11111111` | apagado (`others`) | | | |

---

*Etapa 2*

## 3. Adaptações de Hardware (DE10-Lite)

**O que a arquitetura original usava:** o código-fonte do livro é apenas a entidade aritmética (`sign1/exp1/frac1`, `sign2/exp2/frac2` → `sign_out/exp_out/frac_out`), sem qualquer interface física. Ele assume que os 26 bits de entrada estão simplesmente disponíveis nos pinos, o que não é possível numa DE10-Lite, que tem **10 chaves deslizantes, 2 botões e 6 displays de 7 segmentos**. Além disso, a normalização original tinha três ramos (carry / "pequeno demais" → zero / normalização convencional) e a saída de sinal era diretamente `big_sign`.

### **O que mudamos no VHDL original:**

* **Removemos** o descarte de resultados que exigiriam expoente negativo. O ramo original
  `elsif (leado > big_exp) then normal_exp <= (others=>'0'); normal_frac <= (others=>'0');`
  zerava o resultado; ele foi substituído por uma **normalização parcial** com `shift_left(frac_sum(7 downto 0), to_integer(big_exp))`, que desloca apenas o que o expoente permite e para em `E = 0`. Sem isso, `0,75 + (−0,5)` retornava `0` em vez de `0,25`.
* **Removemos** a possibilidade de zero negativo: `sign_out <= big_sign;` virou `sign_out <= '0' when normal_frac = "00000000" else big_sign;`.
* **Adicionamos** a detecção explícita de cancelamento exato (`if frac_sum = "000000000"`) **antes** do teste de carry, garantindo `0 | 0000 | 00000000` mesmo quando o cancelamento ocorre com expoentes altos (ex.: `128 + (−128)`), em vez de deixar um expoente residual produzido pelo `leado = 7` do codificador de prioridade.
* **Renomeamos** a entidade do núcleo para `fp_adder_v2`, para distinguir claramente a versão modificada da versão-fonte.
* **Roteamos** as entradas para os recursos físicos da placa por meio de um wrapper novo (`fp_adder_de10`), que é o top-level da síntese: `SW9` seleciona qual metade do operando está sendo carregada, `SW7..SW0` carregam `frac`, `SW4` carrega o sinal, `SW3..SW0` carregam o expoente, `KEY0`/`KEY1` gravam o operando 1/2 e `SW8` é o reset assíncrono.
* **Roteamos** as saídas para os displays através de quatro instâncias de `hex_to_7seg` (módulo novo, também criado para a adaptação): `HEX3` = sinal, `HEX2` = expoente, `HEX1` = `frac(7 downto 4)`, `HEX0` = `frac(3 downto 0)`. `HEX4` e `HEX5` são forçados a `"11111111"` (apagados, pois o display é active-low).
* **Reorganizamos** a entrada de dados em **duas etapas por operando**, já que 2 × 13 = 26 bits não cabem em 10 chaves. Os mesmos switches são reutilizados conforme o modo selecionado por `SW9`.
* **Reorganizamos** a exibição com uma cadeia de prioridade e sinais de *preview*, para que o usuário consiga conferir o operando **antes** de gravá-lo: segurando `KEY0` vê-se o operando 1, segurando `KEY1` vê-se o operando 2, com os dois soltos vê-se o resultado, e com `SW8 = 1` vê-se zero.
* **Adicionamos** um testbench de integração (`tb_fp_adder_de10.vhd`) que estimula o circuito **apenas** por `SW` e `KEY` e verifica os padrões de segmentos em `HEX5..HEX0` com `assert`, provando que a adaptação de I/O não alterou a matemática.

### **Descrição gráfica do sistema (atualização do item 2)**

A descrição do núcleo em §2.3–§2.8 permanece válida. A camada adicionada pela adaptação é descrita abaixo.

**Mapeamento lógico dos recursos da placa:**

| Recurso | Função |
|---|---|
| `SW9` | Seletor de campo: `0` = carrega `frac`; `1` = carrega sinal e expoente |
| `SW8` | Reset assíncrono, ativo em nível alto (zera `sign1/exp1/frac1` e `sign2/exp2/frac2`) |
| `SW7..SW0` | Oito bits de `frac` quando `SW9 = 0` |
| `SW4` | Bit de sinal quando `SW9 = 1` |
| `SW3..SW0` | Expoente de 4 bits quando `SW9 = 1` |
| `KEY0` | Grava o campo selecionado no **operando 1** (borda de subida) |
| `KEY1` | Grava o campo selecionado no **operando 2** (borda de subida) |
| `HEX3` | Sinal exibido como `0` ou `1` (`sign_digit <= "000" & display_sign`) |
| `HEX2` | Expoente em hexadecimal |
| `HEX1` / `HEX0` | Nibble alto / baixo da significância |
| `HEX4` / `HEX5` | Permanentemente apagados |

> Os botões da DE10-Lite são **ativos em nível baixo** (`1` = solto, `0` = pressionado). Como o código usa `rising_edge(KEY(n))`, a gravação acontece **na liberação** do botão.

**Cadeia de prioridade da exibição:**

| Prioridade | Condição | O que aparece em `HEX3..HEX0` |
|---|---|---|
| 1 | `SW8 = '1'` | zero (`0000`) |
| 2 | `KEY(0) = '0'` | preview do operando 1 |
| 3 | `KEY(1) = '0'` | preview do operando 2 |
| 4 | caso contrário | resultado da soma |

**Composição do preview** (permite conferir os 13 bits mesmo com carregamento em duas etapas):

| Modo | Sinal | Expoente | Significância |
|---|---|---|---|
| `SW9 = 0` | armazenado | armazenado | **atual em `SW7..SW0`** |
| `SW9 = 1` | **atual em `SW4`** | **atual em `SW3..SW0`** | armazenado |

**Configuração de síntese (`de10_lite_final.qsf`):**

| Item | Valor |
|---|---|
| Top-level entity | `fp_adder_de10` |
| Dispositivo | MAX 10 `10M50DAF484C7G` |
| `KEY0` | `PIN_B8` |
| `KEY1` | `PIN_A7` |
| `SW8` | `PIN_B14` |
| `SW9` | `PIN_F15` |
| Padrão de I/O — `SW`, `HEX` | 3.3-V LVTTL |
| Padrão de I/O — `KEY` | 3.3 V Schmitt Trigger |

---

## 4. Evidências de Validação

### Simulação

A validação foi feita em dois níveis, ambos com **GHDL** e visualização no **GTKWave**:

1. `tb_fp_adder.vhd` — estimula o núcleo `fp_adder_v2` diretamente (sem switches/botões/displays); forma de onda em `adder_wave.vcd`;
2. `tb_fp_adder_de10.vhd` — estimula o sistema completo **apenas** por `SW` e `KEY` e verifica os padrões de segmentos; forma de onda em `wave.vcd`.

Comandos utilizados:

```bash
# análise
ghdl -a final_fp_adder.vhd
ghdl -a hex_to_7seg.vhd
ghdl -a final_fp_adder_de10.vhd
ghdl -a tb_fp_adder.vhd
ghdl -a tb_fp_adder_de10.vhd

# elaboração e execução do testbench do núcleo
ghdl -e tb_fp_adder
ghdl -r tb_fp_adder --vcd=adder_wave.vcd

# elaboração e execução do testbench de integração
ghdl -e tb_fp_adder_de10
ghdl -r tb_fp_adder_de10 --vcd=wave.vcd

# visualização
gtkwave adder_wave.vcd
gtkwave wave.vcd
```

#### Casos de teste e resultados esperados

Os dois testbenches usam os **mesmos sete casos**, escolhidos para cobrir exatamente os quatro casos do 4º estágio (normalização). Todas as verificações são feitas com `assert ... severity error`.

| # | Operação | Entradas (`S \| E \| F`) | Saída esperada | Display | Caso de normalização exercitado |
|---|---|---|---|---|---|
| 1 | `0,75 + (−0,5) = 0,25` | `0\|0000\|11000000` e `1\|0000\|10000000` | `0\|0000\|01000000` | `0040` | **Caso 3** (parcial, `leado=1 > big_exp=0`) |
| 2 | `0,25 + 0 = 0,25` | `0\|0001\|00100000` e `0\|0000\|00000000` | `0\|0000\|01000000` | `0040` | **Caso 3** (`shift_left` por `big_exp = 1`) |
| 3 | `0,25 + 0,25 = 0,5` | `0\|0000\|01000000` (×2) | `0\|0000\|10000000` | `0080` | **Caso 4** (convencional, `leado = 0`) |
| 4 | `0,00390625 + 0,00390625 = 0,0078125` | `0\|0000\|00000001` (×2) | `0\|0000\|00000010` | `0002` | **Caso 3** (limite inferior) |
| 5 | `+0,00390625 + (−0,0078125) = −0,00390625` | `0\|0000\|00000001` e `1\|0000\|00000010` | `1\|0000\|00000001` | `1001` | **Caso 3** + preservação do sinal negativo |
| 6 | `+0,00390625 + (−0,00390625) = 0` | `0\|0000\|00000001` e `1\|0000\|00000001` | `0\|0000\|00000000` | `0000` | **Caso 1** (zero canônico) |
| 7 | `16320 + 16320 = 32640` | `0\|1110\|11111111` (×2) | `0\|1111\|11111111` | `0FFF` | **Caso 2** (carry, `frac_sum(8) = '1'`) |

Os quatro casos do 4º estágio pedidos na Etapa 1, detalhados:

* **Caso 1 — cancelamento exato (Teste 6):** `frac_sum = "000000000"`; expoente e significância são zerados e `sign_out` é forçado a `'0'`. Verifica-se que não sobra expoente residual nem aparece `−0`.
* **Caso 2 — carry (Teste 7):** `0_11111111 + 0_11111111 = 1_11111110`; como `frac_sum(8) = '1'`, o resultado é deslocado uma casa à direita (`frac_sum(8 downto 1) = 11111111`) e o expoente vai de `1110` para `1111`.
* **Caso 3 — normalização parcial (Testes 1, 2, 4 e 5):** `leado > big_exp`. No Teste 2, `frac_sum = 0_00100000` com `leado = 2` e `big_exp = 1`: como só há uma unidade de expoente disponível, o circuito desloca **uma** casa (`shift_left(..., 1) = 01000000`) e para com `E = 0`. O valor 0,25 é preservado, ainda que a significância não fique normalizada.
* **Caso 4 — normalização convencional (Teste 3):** `frac_sum = 0_10000000`, `leado = 0`, `big_exp = 0`; como `leado` não é maior que `big_exp`, aplica-se `normal_exp = big_exp − leado` e `normal_frac = normal_sum`, devolvendo a forma `0.1xxxxxxx`.

#### Testbenches de visualização das formas de onda

Para produzir formas de onda legíveis do 4º estágio, foram escritos dois testbenches adicionais que **não alteram nenhum arquivo do projeto** — apenas instanciam os módulos existentes:

| Arquivo | Alvo | Casos | Duração |
|---|---|---|---|
| `tb_ondas_core.vhd` | `fp_adder_v2` diretamente | 8 | 100 ns por caso, 800 ns total |
| `tb_ondas_de10.vhd` | `fp_adder_de10` via `SW`/`KEY` | 9 | 820 ns por caso, 7950 ns total |

O segundo reproduz exatamente o procedimento manual usado na placa (reset por `SW8`, fração com `SW9 = 0`, sinal/expoente com `SW9 = 1`, gravação na borda de subida de `KEY0`/`KEY1`) e decodifica os segmentos de volta para dígitos (`disp3..disp0`), de modo que a onda mostra diretamente `0 0 4 0` em vez de padrões de oito bits. Um sinal `fase` indica em que etapa do protocolo a simulação está.

Além dos casos da tabela anterior, esses testbenches cobrem dois comportamentos observados no teste presencial:

| # | Operação | Resultado | Explicação |
|---|---|---|---|
| T7 | `32000 + 100` | `0FFA` (= 32000) | `diff_exp = 15 − 7 = 8` → `align_frac = "00000000"`: o menor operando é descartado no alinhamento. Coerente com a resolução do formato, que em `E = 15` é de 128 |
| T8 | `32640 + 32640` | `00FF` (= 0,996) | **Overflow do expoente:** `big_exp = 1111` e `+1` faz wrap para `0000`. Limitação documentada, sem saturação nem flag |

**Formas de onda:**

Visão geral dos oito casos do núcleo (`ondas_core.vcd`, 0 a 800 ns):

![Visão geral - núcleo](<img width="1600" height="825" alt="image" src="https://github.com/user-attachments/assets/0ff9791c-ea9c-45a6-924a-a765dd220f92" />)

Os quatro casos da normalização, em detalhe:

| Caso | Janela | Figura |
|---|---|---|
| Caso 4 — normalização convencional | 0–100 ns | `![Caso 4](fig-caso4.png)` |
| Caso 3 — normalização parcial | 100–200 ns | `![Caso 3a](fig-caso3a.png)` |
| Caso 3 — `shift_left` limitado por `big_exp` | 200–300 ns | `![Caso 3b](fig-caso3b.png)` |
| Caso 1 — cancelamento / zero canônico | 300–400 ns | `![Caso 1](fig-caso1.png)` |
| Caso 2 — carry | 400–500 ns | `![Caso 2](fig-caso2.png)` |

Protocolo de entrada pelas chaves e resultado nos displays (`ondas_de10.vcd`):
 
## Testbench do Sistema Completo — `tb_ondas_de10.vhd`, sendo referente a como a placa esta reagindo frente aos inputs que ela oferece
 
---
 
### **Teste 1: Visão Geral — Protocolo Completo em 9 Casos**
 
**Intervalo:** 0 a 7950 ns (Zoom Fit)
 
**O que testa:** Sequência completa do protocolo DE10-Lite (chaves, botões, displays) em todos os 9 casos, mostrando `SW8/SW9`, `KEY0/KEY1`, `fase`, e os displays hexadecimais `HEX3..HEX0` em tempo real.
 
![Visão geral do protocolo DE10-Lite](<img width="1600" height="924" alt="image" src="https://github.com/user-attachments/assets/a12e7e7e-1e30-4c36-8e36-480118446104" />)
 
---
 
### **Teste 1.1: Protocolo Completo — Caso T1 (0,25 + 0,25 = 0,5)**
 
**Intervalo:** 0 a 820 ns
 
**O que testa:** Ciclo completo de entrada: reset → fração operando 1 → sinal/expoente operando 1 → fração operando 2 → sinal/expoente operando 2 → resultado nos displays. Resultado: `0080` (0,5).
 
![Protocolo completo T1](<img width="1404" height="948" alt="image" src="https://github.com/user-attachments/assets/ab25c284-ec29-4b73-95fb-d9bcdde48210" />)
 
---
 
### **Teste 1.2: Protocolo — Caso T2 (0,75 + (−0,5) = 0,25)**
 
**Intervalo:** 820 a 1640 ns
 
**O que testa:** Normalização parcial através do protocolo de chaves. Displays mostram `0040` (0,25).
 
![Protocolo T2 — normalização parcial](<img width="1419" height="944" alt="image" src="https://github.com/user-attachments/assets/07e85081-62a9-4281-815f-c29560a3ace1" />)
 
---
 
### **Teste 1.3: Protocolo — Caso T3 (0,25 não normalizado + 0)**
 
**Intervalo:** 1640 a 2460 ns
 
**O que testa:** `shift_left` limitado por `big_exp`. Displays mostram `0040` (0,25).
 
![Protocolo T3 — shift_left limitado](<img width="1600" height="956" alt="image" src="https://github.com/user-attachments/assets/7c761f45-1282-4287-9558-2d269d270168" />)
 
---
 
### **Teste 1.4: Protocolo — Caso T4 (1,5 + (−1,5) = 0)**
 
**Intervalo:** 2460 a 3280 ns
 
**O que testa:** Cancelamento exato e zero canônico. Displays mostram `0000`.
 
![Protocolo T4 — cancelamento, zero canônico](<img width="1600" height="963" alt="image" src="https://github.com/user-attachments/assets/0e8b0572-f999-4377-aac2-fde5680ff17f" />)
 
---
 
### **Teste 1.5: Protocolo — Caso T5 (16320 + 16320 = 32640)**
 
**Intervalo:** 3280 a 4100 ns
 
**O que testa:** Carry na soma. Displays mostram `0FFF` (32640).
 
![Protocolo T5 — carry, exp+1](<img width="1600" height="966" alt="image" src="https://github.com/user-attachments/assets/a8c0ab02-537b-4bb0-8e76-6569e6b71e90" />)
 
---
 
### **Teste 1.6: Protocolo — Caso T6 (1,0 + (−1,5) = −0,5)**
 
**Intervalo:** 4100 a 4920 ns
 
**O que testa:** Resultado negativo. O primeiro dígito hexadecimal (`HEX3`) mostra `1` (sinal negativo). Displays mostram `1080`.
 
![Protocolo T6 — resultado negativo](<img width="1600" height="970" alt="image" src="https://github.com/user-attachments/assets/58b4b3c2-af91-47be-b01d-2f2ab5bfb4a5" />)
 
---
 
### **Teste 1.7: Protocolo — Caso T7 (32000 + 100 = 32000 — Truncamento)**
 
**Intervalo:** 4920 a 5740 ns
 
**O que testa:** Limitação: `diff_exp = 8` causa descarte completo do menor operando. Displays mostram `0FFA` (32000). Este é o caso observado na demonstração presencial.
 
![Protocolo T7 — truncamento, diff_exp ≥ 8](<img width="1600" height="965" alt="image" src="https://github.com/user-attachments/assets/f14dc06a-24ab-4587-b956-48efa25946ee" />)
 
---
 
### **Teste 1.8: Protocolo — Caso T8 (32640 + 32640 — Overflow do Expoente)**
 
**Intervalo:** 5740 a 6560 ns
 
**O que testa:** Limitação documentada: overflow do expoente sem tratamento. Displays mostram `00FF` (resultado incorreto, devido ao wrap-around de `exp 1111 + 1 = 0000`).
 
![Protocolo T8 — overflow do expoente](<img width="1600" height="969" alt="image" src="https://github.com/user-attachments/assets/ed76b0fc-8ccc-4f77-80ed-36cc436f07f4" />)
 
---
 
### **Teste 1.9: Preview — Demonstração de Operandos Armazenados**
 
**Intervalo:** 6560 a 7950 ns
 
**O que testa:** Funcionalidade de preview: enquanto `KEY0` é pressionado, o display mostra o operando 1 armazenado; enquanto `KEY1` é pressionado, mostra o operando 2; com ambos soltos, mostra o resultado. Cargas: operando 1 = 1,5 (`01C0`), operando 2 = 0,5 (`0080`), resultado = 2,0 (`0280`).
 
![Preview — KEY0 = operando 1](<img width="1600" height="966" alt="image" src="https://github.com/user-attachments/assets/a3b52c0a-325b-456a-9cfa-8658878c5cc1" />)
 
---
 
## Resumo
 
| # | Teste | Intervalo | Operação | Resultado Esperado |
|---|---|---|---|---|
| 1 | Visão geral núcleo | 0–800 ns | Todos os 8 casos | — |
| 2 | Caso 4 | 0–100 ns | 0,25 + 0,25 | 0,5 (`0080`) |
| 3 | Caso 3a | 100–200 ns | 0,75 + (−0,5) | 0,25 (`0040`) |
| 4 | Caso 3b | 200–300 ns | 0,00100000×2¹ + 0 | 0,25 (`0040`) |
| 5 | Caso 1 | 300–400 ns | 1,5 + (−1,5) | 0 (`0000`) |
| 6 | Caso 2 | 400–500 ns | 16320 + 16320 | 32640 (`0FFF`) |
| 7 | Negativo | 500–600 ns | 1,0 + (−1,5) | −0,5 (`1080`) |
| 8 | Truncamento | 600–700 ns | 32000 + 100 | 32000 (`0FFA`) |
| 9 | Overflow | 700–800 ns | 32640 + 32640 | `00FF` (limitação) |
| 10 | Visão geral DE10 | 0–7950 ns | Protocolo completo | — |
| 11–18 | Protocolo T1–T8 | 820 ns/caso | Idem acima | Displays na placa |
| 19 | Preview | 6560–7950 ns | Demonstração | Operandos armazenados |

### Código VHDL Final

#### 4.1 Trechos centrais da adaptação

**(a) Núcleo — processo de normalização com os quatro casos.** É aqui que estão as três correções em relação ao código original; os Casos 1 e 3 são as mudanças de comportamento mais importantes do projeto.

```vhdl
-- normalize with special conditions --
process(frac_sum, normal_sum, big_exp, leado)
begin
    -- Caso 1 [ADICIONADO]: cancelamento completo -> zero canônico
    if frac_sum = "000000000" then
        normal_exp  <= (others => '0');
        normal_frac <= (others => '0');

    -- Caso 2 [ORIGINAL]: carry na soma -> desloca 1 à direita e incrementa o expoente
    elsif frac_sum(8) = '1' then
        normal_exp  <= big_exp + 1;
        normal_frac <= frac_sum(8 downto 1);

    -- Caso 3 [MODIFICADO]: 4 bits de expoente não bastam.
    -- Original: zerava o resultado. Agora: normalização parcial até E = 0,
    -- preservando valores menores que 0,5.
    elsif leado > big_exp then
        normal_exp  <= (others => '0');
        normal_frac <= shift_left(
            frac_sum(7 downto 0),
            to_integer(big_exp)
        );

    -- Caso 4 [ORIGINAL]: normalização convencional
    else
        normal_exp  <= big_exp - leado;
        normal_frac <= normal_sum;
    end if;
end process;
```

**(b) Núcleo — zero canônico na saída.** Sem esta linha, um cancelamento em que o operando negativo caiu no ramo `else` da ordenação produziria `−0`.

```vhdl
sign_out <= '0' when normal_frac = "00000000" else big_sign;
```

**(c) Wrapper — carregamento em duas etapas com reset assíncrono.** Resolve a limitação de 10 chaves para 26 bits de entrada.

```vhdl
process(KEY(0), SW(8))
begin
    if SW(8) = '1' then                 -- reset assíncrono
        sign1 <= '0';
        exp1  <= "0000";
        frac1 <= "00000000";
    elsif rising_edge(KEY(0)) then      -- grava na liberação do botão
        if SW(9) = '0' then
            frac1 <= SW(7 downto 0);    -- etapa 1: significância
        else
            sign1 <= SW(4);             -- etapa 2: sinal...
            exp1  <= SW(3 downto 0);    -- ...e expoente
        end if;
    end if;
end process;
```

**(d) Wrapper — preview do operando em edição.** Combina o campo que está nas chaves com o campo já armazenado.

```vhdl
preview1_sign <= sign1          when SW(9) = '0' else SW(4);
preview1_exp  <= exp1           when SW(9) = '0' else SW(3 downto 0);
preview1_frac <= SW(7 downto 0) when SW(9) = '0' else frac1;
```

**(e) Wrapper — cadeia de prioridade da exibição.**

```vhdl
display_sign <=
    '0'           when SW(8)  = '1' else   -- reset
    preview1_sign when KEY(0) = '0' else   -- segurando KEY0
    preview2_sign when KEY(1) = '0' else   -- segurando KEY1
    sign_out;                              -- resultado
```

**(f) Wrapper — roteamento para os displays.** O sinal vira um nibble para reaproveitar o mesmo decodificador; `HEX4`/`HEX5` ficam apagados (active-low).

```vhdl
sign_digit <= "000" & display_sign;

display_sign_decoder:      entity work.hex_to_7seg
    port map (value => sign_digit,                segments => HEX3);
display_exp_decoder:       entity work.hex_to_7seg
    port map (value => display_exp,               segments => HEX2);
display_frac_high_decoder: entity work.hex_to_7seg
    port map (value => display_frac(7 downto 4),  segments => HEX1);
display_frac_low_decoder:  entity work.hex_to_7seg
    port map (value => display_frac(3 downto 0),  segments => HEX0);

HEX4 <= "11111111";
HEX5 <= "11111111";
```

#### 4.2 Código completo

<details>
<summary><b>final_fp_adder.vhd</b> — núcleo aritmético (<code>fp_adder_v2</code>)</summary>

```vhdl
library ieee;
use ieee.std_logic_1164.all;
use ieee.numeric_std.all;

entity fp_adder_v2 is
	port(
	sign1: in std_logic;
	exp1: in std_logic_vector(3 downto 0);
	frac1: in std_logic_vector(7 downto 0);

	sign2: in std_logic;
	exp2: in std_logic_vector(3 downto 0);
	frac2: in std_logic_vector(7 downto 0);

	sign_out: out std_logic;
	exp_out: out std_logic_vector(3 downto 0);
	frac_out: out std_logic_vector(7 downto 0)
	);
end fp_adder_v2;

architecture arch of fp_adder_v2 is
	signal big_sign, small_sign: std_logic;
	signal big_exp, small_exp, normal_exp: unsigned (3 downto 0);
	signal big_frac, small_frac, align_frac, normal_frac: unsigned (7 downto 0);
	signal normal_sum: unsigned (7 downto 0);
	signal diff_exp: unsigned (3 downto 0);
	signal frac_sum: unsigned (8 downto 0); -- Um extra para carry --
	signal leado: unsigned (2 downto 0);
begin
	--1st stage: sort to find the largest number --
	process(
	sign1, sign2, exp1, exp2, frac1, frac2
	)
	begin
		if (exp1 & frac1) > (exp2 & frac2) then
			big_sign <= sign1;
			small_sign <= sign2;
			big_exp <= unsigned(exp1);
			small_exp <= unsigned(exp2);
			big_frac <= unsigned(frac1);
			small_frac <= unsigned(frac2);
		else
			big_sign <= sign2;
			small_sign <= sign1;
			big_exp <= unsigned(exp2);
			small_exp <= unsigned(exp1);
			big_frac <= unsigned(frac2);
			small_frac <= unsigned(frac1);
		end if;
	end process;

	-- 2nd stage: align smaller number --
	diff_exp <= big_exp - small_exp;
	with diff_exp select
		align_frac <=
		small_frac when "0000",
		"0" & small_frac(7 downto 1) when "0001",
		"00" & small_frac(7 downto 2) when "0010",
		"000" & small_frac(7 downto 3) when "0011",
		"0000" & small_frac(7 downto 4) when "0100",
		"00000" & small_frac(7 downto 5) when "0101",
		"000000" & small_frac(7 downto 6) when "0110",
		"0000000" & small_frac(7) when "0111",
		"00000000" when others;

	-- 3rd stage: add/subtract --
	frac_sum <=
		('0' & big_frac) + ('0' & align_frac) when big_sign=small_sign else
		('0' & big_frac) - ('0' & align_frac);

	-- 4th stage: normalize --
	-- count leading 0s --
	leado <=
	"000" when (frac_sum(7) = '1') else
	"001" when (frac_sum(6) = '1') else
	"010" when (frac_sum(5) = '1') else
	"011" when (frac_sum(4) = '1') else
	"100" when (frac_sum(3) = '1') else
	"101" when (frac_sum(2) = '1') else
	"110" when (frac_sum(1) = '1') else
	"111";

	-- shift significand according to leading 0 --
	with leado select
		normal_sum <=
		frac_sum(7 downto 0) when "000",
		frac_sum(6 downto 0) & '0' when "001",
		frac_sum(5 downto 0) & "00"  when "010",
		frac_sum(4 downto 0) & "000" when "011",
		frac_sum(3 downto 0) & "0000" when "100",
		frac_sum(2 downto 0) & "00000" when "101",
		frac_sum(1 downto 0) & "000000" when "110",
		frac_sum(0) & "0000000" when others;

    -- normalize with special conditions --
    process(
        frac_sum, normal_sum, big_exp, leado
    )
    begin
        -- Caso 1: cancelamento completo: soma = sub -> normaliza para zero
        if frac_sum = "000000000" then
            normal_exp <= (others => '0');
            normal_frac <= (others => '0');
        -- Caso 2: ocorreu carry na soma: frac_sum leq 1.0 -> desloca para direita 1 vez -> ++normal_exp
        elsif frac_sum(8) = '1' then
            normal_exp <= big_exp + 1;
            normal_frac <= frac_sum(8 downto 1);
        -- Caso 3: 4 bits de exp nao e o suficiente: desloca normal_exp para direita, ate 0 -> mantem valores menores que 0.5
        elsif leado > big_exp then
            normal_exp <= (others => '0');
            normal_frac <= shift_left(
                frac_sum(7 downto 0),
                to_integer(big_exp)
            );
        -- Caso 4: 4 bits e o suficiente -> normalizacao convencional baseada no codigo fonte
        else
            normal_exp <= big_exp - leado;
            normal_frac <= normal_sum;
        end if;
    end process;

    -- output --
    sign_out <= '0' when normal_frac = "00000000" else big_sign;
    exp_out <= std_logic_vector(normal_exp);
    frac_out <= std_logic_vector(normal_frac);
end arch;
```

</details>

<details>
<summary><b>hex_to_7seg.vhd</b> — decodificador de 7 segmentos</summary>

```vhdl
library ieee;
use ieee.std_logic_1164.all;

entity hex_to_7seg is
    port (
        value    : in  std_logic_vector(3 downto 0);
        segments : out std_logic_vector(7 downto 0)
    );
end hex_to_7seg;

architecture arch of hex_to_7seg is
begin

    -- The DE10-Lite display uses active-low signals:
    -- 0 turns a segment on
    -- 1 turns a segment off
    --
    -- Bit 7 is the decimal point, which remains off.

    with value select
        segments <=
            "11000000" when "0000", -- 0
            "11111001" when "0001", -- 1
            "10100100" when "0010", -- 2
            "10110000" when "0011", -- 3
            "10011001" when "0100", -- 4
            "10010010" when "0101", -- 5
            "10000010" when "0110", -- 6
            "11111000" when "0111", -- 7
            "10000000" when "1000", -- 8
            "10010000" when "1001", -- 9
            "10001000" when "1010", -- A
            "10000011" when "1011", -- b
            "11000110" when "1100", -- C
            "10100001" when "1101", -- d
            "10000110" when "1110", -- E
            "10001110" when "1111", -- F
            "11111111" when others; -- blank

end arch;
```

</details>

<details>
<summary><b>final_fp_adder_de10.vhd</b> — wrapper / top-level da DE10-Lite</summary>

```vhdl
library ieee;
use ieee.std_logic_1164.all;

entity fp_adder_de10 is
    port (
        SW  : in std_logic_vector(9 downto 0);
        KEY : in std_logic_vector(1 downto 0);

        HEX0 : out std_logic_vector(7 downto 0);
        HEX1 : out std_logic_vector(7 downto 0);
        HEX2 : out std_logic_vector(7 downto 0);
        HEX3 : out std_logic_vector(7 downto 0);
        HEX4 : out std_logic_vector(7 downto 0);
        HEX5 : out std_logic_vector(7 downto 0)
    );
end fp_adder_de10;


architecture arch of fp_adder_de10 is

    -- Stored operand 1
    signal sign1 : std_logic;
    signal exp1  : std_logic_vector(3 downto 0);
    signal frac1 : std_logic_vector(7 downto 0);

    -- Stored operand 2
    signal sign2 : std_logic;
    signal exp2  : std_logic_vector(3 downto 0);
    signal frac2 : std_logic_vector(7 downto 0);

    -- Adder result
    signal sign_out : std_logic;
    signal exp_out  : std_logic_vector(3 downto 0);
    signal frac_out : std_logic_vector(7 downto 0);

    -- Values shown on the displays
    signal display_sign : std_logic;
    signal display_exp  : std_logic_vector(3 downto 0);
    signal display_frac : std_logic_vector(7 downto 0);

    signal sign_digit : std_logic_vector(3 downto 0);

    -- Preview of operand 1
    signal preview1_sign : std_logic;
    signal preview1_exp  : std_logic_vector(3 downto 0);
    signal preview1_frac : std_logic_vector(7 downto 0);

    -- Preview of operand 2
    signal preview2_sign : std_logic;
    signal preview2_exp  : std_logic_vector(3 downto 0);
    signal preview2_frac : std_logic_vector(7 downto 0);

begin

    -- Store operand 1
    -- SW9 = 0: load fraction
    -- SW9 = 1: load sign and exponent
    -- SW8 = 1: reset
    process(KEY(0), SW(8))
    begin
        if SW(8) = '1' then
            sign1 <= '0';
            exp1  <= "0000";
            frac1 <= "00000000";
        elsif rising_edge(KEY(0)) then
            if SW(9) = '0' then
                frac1 <= SW(7 downto 0);
            else
                sign1 <= SW(4);
                exp1  <= SW(3 downto 0);
            end if;
        end if;
    end process;


    -- Store operand 2
    process(KEY(1), SW(8))
    begin
        if SW(8) = '1' then
            sign2 <= '0';
            exp2  <= "0000";
            frac2 <= "00000000";
        elsif rising_edge(KEY(1)) then
            if SW(9) = '0' then
                frac2 <= SW(7 downto 0);
            else
                sign2 <= SW(4);
                exp2  <= SW(3 downto 0);
            end if;
        end if;
    end process;


    -- Preview of operand 1
    preview1_sign <= sign1 when SW(9) = '0' else SW(4);
    preview1_exp  <= exp1  when SW(9) = '0' else SW(3 downto 0);
    preview1_frac <= SW(7 downto 0) when SW(9) = '0' else frac1;

    -- Preview of operand 2
    preview2_sign <= sign2 when SW(9) = '0' else SW(4);
    preview2_exp  <= exp2  when SW(9) = '0' else SW(3 downto 0);
    preview2_frac <= SW(7 downto 0) when SW(9) = '0' else frac2;


    -- Floating-point adder
    adder: entity work.fp_adder_v2
        port map (
            sign1 => sign1,
            exp1  => exp1,
            frac1 => frac1,

            sign2 => sign2,
            exp2  => exp2,
            frac2 => frac2,

            sign_out => sign_out,
            exp_out  => exp_out,
            frac_out => frac_out
        );


    -- Select what appears on the displays
    -- Reset active: display zero
    -- KEY0 held: preview operand 1
    -- KEY1 held: preview operand 2
    -- Otherwise: display the result
    display_sign <=
        '0'           when SW(8) = '1' else
        preview1_sign when KEY(0) = '0' else
        preview2_sign when KEY(1) = '0' else
        sign_out;

    display_exp <=
        "0000"       when SW(8) = '1' else
        preview1_exp when KEY(0) = '0' else
        preview2_exp when KEY(1) = '0' else
        exp_out;

    display_frac <=
        "00000000"    when SW(8) = '1' else
        preview1_frac when KEY(0) = '0' else
        preview2_frac when KEY(1) = '0' else
        frac_out;


    -- Convert the displayed value to hexadecimal
    sign_digit <= "000" & display_sign;

    display_sign_decoder: entity work.hex_to_7seg
        port map (value => sign_digit, segments => HEX3);

    display_exp_decoder: entity work.hex_to_7seg
        port map (value => display_exp, segments => HEX2);

    display_frac_high_decoder: entity work.hex_to_7seg
        port map (value => display_frac(7 downto 4), segments => HEX1);

    display_frac_low_decoder: entity work.hex_to_7seg
        port map (value => display_frac(3 downto 0), segments => HEX0);


    -- Unused displays
    HEX4 <= "11111111";
    HEX5 <= "11111111";

end arch;
```

</details>

<details>
<summary><b>tb_fp_adder_de10.vhd</b> — testbench de integração (protocolo SW/KEY + verificação dos displays)</summary>

```vhdl
library ieee;
use ieee.std_logic_1164.all;

entity tb_fp_adder_de10 is
end entity;

architecture sim of tb_fp_adder_de10 is

    -- Entradas da DE10-Lite
    signal SW  : std_logic_vector(9 downto 0) := (others => '0');
    signal KEY : std_logic_vector(1 downto 0) := "11";

    -- Displays
    signal HEX0 : std_logic_vector(7 downto 0);
    signal HEX1 : std_logic_vector(7 downto 0);
    signal HEX2 : std_logic_vector(7 downto 0);
    signal HEX3 : std_logic_vector(7 downto 0);
    signal HEX4 : std_logic_vector(7 downto 0);
    signal HEX5 : std_logic_vector(7 downto 0);

    -- Padroes do display de 7 segmentos (active-low)
    constant SEG_0   : std_logic_vector(7 downto 0) := "11000000";
    constant SEG_1   : std_logic_vector(7 downto 0) := "11111001";
    constant SEG_2   : std_logic_vector(7 downto 0) := "10100100";
    constant SEG_4   : std_logic_vector(7 downto 0) := "10011001";
    constant SEG_8   : std_logic_vector(7 downto 0) := "10000000";
    constant SEG_F   : std_logic_vector(7 downto 0) := "10001110";
    constant SEG_OFF : std_logic_vector(7 downto 0) := "11111111";

    -- Gera uma borda de subida no botao
    procedure pulse_key(signal key_n : out std_logic) is
    begin
        key_n <= '0';
        wait for 5 ns;
        key_n <= '1';
        wait for 5 ns;
    end procedure;

    -- Carrega um operando no wrapper (fracao e depois sinal/expoente)
    procedure load_operand(
        signal sw_bus : out std_logic_vector(9 downto 0);
        signal key_n  : out std_logic;
        constant sign_v : in std_logic;
        constant exp_v  : in std_logic_vector(3 downto 0);
        constant frac_v : in std_logic_vector(7 downto 0)
    ) is
    begin
        sw_bus(9) <= '0';
        sw_bus(7 downto 0) <= frac_v;
        pulse_key(key_n);

        sw_bus(9) <= '1';
        sw_bus(4) <= sign_v;
        sw_bus(3 downto 0) <= exp_v;
        pulse_key(key_n);
    end procedure;

    -- Verifica os quatro displays usados pelo resultado
    procedure check_display(
        constant test_name     : in string;
        constant expected_hex3 : in std_logic_vector(7 downto 0);
        constant expected_hex2 : in std_logic_vector(7 downto 0);
        constant expected_hex1 : in std_logic_vector(7 downto 0);
        constant expected_hex0 : in std_logic_vector(7 downto 0)
    ) is
    begin
        assert (
            HEX3 = expected_hex3 and
            HEX2 = expected_hex2 and
            HEX1 = expected_hex1 and
            HEX0 = expected_hex0
        )
        report "ERRO " & test_name & ": HEX3..HEX0 incorretos"
        severity error;

        assert (HEX4 = SEG_OFF and HEX5 = SEG_OFF)
        report "ERRO " & test_name & ": HEX4 e/ou HEX5 deveriam estar apagados"
        severity error;
    end procedure;

begin

    dut: entity work.fp_adder_de10
        port map (
            SW  => SW,
            KEY => KEY,
            HEX0 => HEX0, HEX1 => HEX1, HEX2 => HEX2,
            HEX3 => HEX3, HEX4 => HEX4, HEX5 => HEX5
        );

    stimulus: process
    begin

        -- TESTE 1: 0.75 + (-0.5) = 0.25  ->  0040
        SW(8) <= '1'; KEY <= "11"; wait for 5 ns;
        SW(8) <= '0'; wait for 5 ns;
        load_operand(SW, KEY(0), '0', "0000", "11000000");
        load_operand(SW, KEY(1), '1', "0000", "10000000");
        wait for 5 ns;
        check_display("TESTE 1", SEG_0, SEG_0, SEG_4, SEG_0);

        -- TESTE 2: normalizacao parcial 0.00100000 * 2^1 = 0.25  ->  0040
        SW(8) <= '1'; KEY <= "11"; wait for 5 ns;
        SW(8) <= '0'; wait for 5 ns;
        load_operand(SW, KEY(0), '0', "0001", "00100000");
        load_operand(SW, KEY(1), '0', "0000", "00000000");
        wait for 5 ns;
        check_display("TESTE 2", SEG_0, SEG_0, SEG_4, SEG_0);

        -- TESTE 3: 0.25 + 0.25 = 0.5  ->  0080
        SW(8) <= '1'; KEY <= "11"; wait for 5 ns;
        SW(8) <= '0'; wait for 5 ns;
        load_operand(SW, KEY(0), '0', "0000", "01000000");
        load_operand(SW, KEY(1), '0', "0000", "01000000");
        wait for 5 ns;
        check_display("TESTE 3", SEG_0, SEG_0, SEG_8, SEG_0);

        -- TESTE 4: menores positivos, 1/256 + 1/256  ->  0002
        SW(8) <= '1'; KEY <= "11"; wait for 5 ns;
        SW(8) <= '0'; wait for 5 ns;
        load_operand(SW, KEY(0), '0', "0000", "00000001");
        load_operand(SW, KEY(1), '0', "0000", "00000001");
        wait for 5 ns;
        check_display("TESTE 4", SEG_0, SEG_0, SEG_0, SEG_2);

        -- TESTE 5: menor resultado negativo  ->  1001
        SW(8) <= '1'; KEY <= "11"; wait for 5 ns;
        SW(8) <= '0'; wait for 5 ns;
        load_operand(SW, KEY(0), '0', "0000", "00000001");
        load_operand(SW, KEY(1), '1', "0000", "00000010");
        wait for 5 ns;
        check_display("TESTE 5", SEG_1, SEG_0, SEG_0, SEG_1);

        -- TESTE 6: cancelamento no menor valor representavel  ->  0000
        SW(8) <= '1'; KEY <= "11"; wait for 5 ns;
        SW(8) <= '0'; wait for 5 ns;
        load_operand(SW, KEY(0), '0', "0000", "00000001");
        load_operand(SW, KEY(1), '1', "0000", "00000001");
        wait for 5 ns;
        check_display("TESTE 6", SEG_0, SEG_0, SEG_0, SEG_0);

        -- TESTE 7: maior resultado sem overflow, 16320 + 16320  ->  0FFF
        SW(8) <= '1'; KEY <= "11"; wait for 5 ns;
        SW(8) <= '0'; wait for 5 ns;
        load_operand(SW, KEY(0), '0', "1110", "11111111");
        load_operand(SW, KEY(1), '0', "1110", "11111111");
        wait for 5 ns;
        check_display("TESTE 7", SEG_0, SEG_F, SEG_F, SEG_F);

        report "Todos os 7 testes do wrapper foram executados." severity note;
        wait;

    end process;

end architecture;
```

</details>

`[PENDENTE: inserir o conteúdo de tb_fp_adder.vhd — o testbench do núcleo]`

---

*Etapa 3*

### Funcionamento na Placa

Procedimento de operação na DE10-Lite:

1. Ligar `SW8` (reset) e conferir `0000` nos displays; desligar `SW8`.
2. `SW9 = 0`, ajustar `SW7..SW0` com a significância do operando 1 e pressionar/soltar `KEY0`.
3. `SW9 = 1`, ajustar `SW4` (sinal) e `SW3..SW0` (expoente) e pressionar/soltar `KEY0`.
4. Repetir os passos 2 e 3 usando `KEY1` para o operando 2.
5. Com os dois botões soltos, os displays mostram o resultado. Não existe botão de "somar": o núcleo é combinacional e o resultado atualiza sozinho.
6. A qualquer momento, segurar `KEY0` ou `KEY1` para conferir o operando correspondente.

Casos críticos a demonstrar (mesmos casos validados em simulação):

| Caso | Operação | Entradas na placa | Display esperado |
|---|---|---|---|
| Carry / normalização à direita | `16320 + 16320` | `0\|1110\|11111111` (×2) | `0FFF` |
| Normalização parcial (`E = 0`) | `0,75 + (−0,5)` | `0\|0000\|11000000` e `1\|0000\|10000000` | `0040` |
| Cancelamento exato (zero canônico) | `x + (−x)` | `0\|0000\|00000001` e `1\|0000\|00000001` | `0000` |
| Resultado negativo | `+0,00390625 + (−0,0078125)` | `0\|0000\|00000001` e `1\|0000\|00000010` | `1001` |

---

*Etapa 4*

## 5. Diário de Bordo de IA

Utilizamos o `[ChatGPT/Claude/Gemini]` para auxiliar na geração do Testbench e na refatoração do código. Abaixo está a análise do uso da ferramenta.

GPT - Utilizado para tirar dúvidas gerais do projetos, com exemplos do que deveriamos alterar no vhdl, todo exemplos falharam, IA não conseguiu gerar o código assim como as outras, sempre exisitam erros e bugs especifícos que demandavam altear maior parte do código, sendoa IA para vhdl não tão efiicente para projetos que demandam maior complexidade, mesmo com descrição do projeto e arquivos anexados, não gerou um resultado interessante.

CLAUDE - Mais eficiente, utilizado para analisar o projeto pronto e ajudar no preenchimento do relatório, tanto na formatação do .md, mas tbm ajudar com a confeção de testes para enriquecimento do mesmo. Foi descrito informalmente para Gemini com código pronto, para fazer um prompt otimziado para ia, descrevendo como os testes foram feitos presecialmente + análise do vhdl, foi possível gerar vários testes rapidamente, após apresentação do projeto presencialmente, foi possivel ter outra forma de validar o projeto (Todos os prints de testes foram analisados, para garantir que não houve alucinação de IA)

histórico em md da conversa gerada para o frontend, calculadora para validar os resultados e nos ajudar nos testes da placa esta documentado e disponíveis para download:


**Prompts Utilizados:**

> `Chat Conversation
Note: This is purely the output of the chat conversation and does not contain any raw data, codebase snippets, etc. used to generate the output.
User Input
quero fazer um frontend de uma pagina pra nos ajudar a testar isso pessoalmente, basicamente front end bonito porem sem framework, sem complicar nada, zero
precisa ter o seguinte
dois inputs de numero decimal, usuario coloca os numeros a serem somados
e é apresentado as instrucoes pra colocar o numero 1 na placa (SW9 até SW0)
o que deve aparecer nos displays depois de apertar key0
as instrucoes para colocar o numero 2
e oque deve aparecer apos apertar key1 (resultado da soma)
o arquivo context.MD tem um resumo do que é o projeto, mas isso é um pouco antigo, o codigo foi alterado, o codigo final ta na pasta final, a mudanca que foi feita foi passar a usar 8 bits para fracao e nao 4
faça o que te pedi e depois atualize o markdown por favor`

**O Erro da IA (Alucinação):**

> `[PENDENTE: descrever os erros observados]`


**A Correção Humana:**

---

## 6. Contribuição dos participantes

Utilize a taxonomia CRediT, seguem exemplos:

* `Angelo Martins Finassi`, Confecção dos relatórios, testes no GTKWave e presenciais 
* `Daniel`, ---
* `Leandro`, Redação do manuscrito original, Validação de dados e experimentos

`[PENDENTE: substituir pelos nomes reais e pelas contribuições efetivas de cada integrante]`

---

## Anexo — Limitações conhecidas do projeto

Documentadas explicitamente para deixar claro o escopo do formato simplificado:

* **Não é IEEE 754:** sem bias, bit implícito, NaN, infinito, subnormais formais, modos de arredondamento ou flags de exceção.
* **Expoente restrito a 0..15,** sem valores negativos. Valores pequenos só são preservados pela região não normalizada em `E = 0`, até `1/256 = 0,00390625`.
* **Sem arredondamento:** bits deslocados para fora no alinhamento são truncados (não há guard/round/sticky bits).
* **`diff_exp ≥ 8` anula o menor operando:** o alinhador só tem casos explícitos de 0 a 7 deslocamentos. **Medido:** `32000 + 100` devolve `32000` (`0FFA`), porque `diff_exp = 8` zera `align_frac`. Note que 100 é menor que o próprio passo de resolução em `E = 15`, que é 128 — o valor 32100 não seria representável de qualquer forma.
* **Overflow do expoente não é tratado:** `normal_exp <= big_exp + 1` com `big_exp = 1111` faz wrap-around para `0000`. Não há saturação em 32640, código de infinito nem flag de erro. **Medido:** `32640 + 32640` devolve `00FF` (0,99609375) em vez de 65280.
* **Precisão cai com a magnitude:** o passo de representação é `2^(E−8)` — 0,00390625 em `E = 0` e 128 em `E = 15`.
* **Entradas não são validadas:** o wrapper aceita qualquer padrão de 13 bits, inclusive codificações não canônicas, para as quais a comparação por `exp & frac` pode não corresponder à comparação real de magnitudes.
* **Botões usados diretamente como evento de gravação,** sem debounce nem sincronização com o clock de 50 MHz da placa; o reset por `SW8` é assíncrono.
* **Núcleo sem pipeline:** todo o caminho (ordenação → alinhamento → soma → contagem de zeros → deslocamento → normalização) é um único caminho combinacional. Aceitável aqui, já que a interação é manual.
* **Os displays mostram campos, não o valor decimal:** `0FFF` significa `S=0`, `E=F`, `F=FF`, e não o inteiro `0x0FFF`.
* **Cobertura de testes não é exaustiva:** 7 casos por testbench, contra 2²⁶ pares possíveis de entradas.
