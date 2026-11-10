# Take-home: firmware

Este exercício faz parte do processo seletivo para engenheiro(a) de firmware da Group Link One. Ele substitui uma bateria de perguntas teóricas e imita dois problemas que resolvemos de verdade: o payload que um medidor manda por BLE e a fila de resultados que uma jig de fábrica guarda quando a rede cai.

Não precisa de hardware. Tudo roda no seu computador.

Prazo: 7 dias corridos a partir do envio. Na entrevista técnica a gente abre o seu código junto com você e conversa sobre as decisões.

## Regras

- C ou C++. Preferimos C++, no padrão mais novo que você conseguir usar.
- IA é permitida e encorajada. Se usar, inclua um `AI.md` curto dizendo onde usou, o que aceitou, o que descartou e como verificou.
- Dúvidas sobre o enunciado: mande para e.barreto@grouplinkone.com. Se preferir seguir sem perguntar, assuma algo razoável e registre a suposição no README.

## Parte 1: codec do payload de advertising

Um medidor anuncia a leitura em 12 bytes dentro do manufacturer data do advertising BLE. Para caber, os campos são empacotados em bits, sem alinhamento a byte.

Os campos entram em sequência a partir do bit 0, na ordem da tabela. O bit `n` do payload é o bit `n % 8` do byte `n / 8`, e cada campo é gravado do bit menos significativo para o mais significativo.

| Bits | Campo | Largura | Observação |
|---|---|---|---|
| 0 a 3 | version | 4 | sempre `1` nesta versão |
| 4 a 11 | device_type | 8 | |
| 12 a 14 | flags | 3 | bit0 bateria baixa, bit1 violação, bit2 fluxo reverso |
| 15 a 41 | pulse_count | 27 | |
| 42 a 53 | battery_mv | 12 | faixa válida 1800 a 3600 |
| 54 a 64 | temperature | 11 | com sinal, complemento de dois, décimos de °C, faixa válida -400 a 850 |
| 65 a 76 | flow | 12 | vazão em L/h, ponto flutuante próprio, descrito abaixo |
| 77 a 84 | sequence | 8 | incrementa a cada anúncio, dá a volta em 255 |
| 85 a 87 | reserved | 3 | zero |
| 88 a 95 | crc | 8 | CRC-8, polinômio `0x07`, init `0x00`, sem reflexão, sobre os bytes 0 a 10 |

O `flow` é um float sem sinal de 12 bits: expoente `E` nos 4 bits altos e mantissa `M` nos 8 bits baixos, com bias 3.

- `E` de 1 a 14: `(1 + M / 256) × 2^(E − 3)`
- `E` igual a 0: `(M / 256) × 2^(−2)`
- `E` igual a 15: inválido

Implemente `encode` e `decode` do payload. Defina a API, os códigos de erro e o que fazer com bits reservados e valores fora da faixa. Escreva as decisões no README.

Vetor de referência: uma leitura com device_type `0x12`, flags bateria baixa e fluxo reverso, pulse_count 123456, battery_mv 3012, temperature -5,7 °C, flow 1200 L/h e sequence 42 vira:

```
21 51 20 F1 00 10 EF F1 59 5A 05 65
```

O CRC de `"123456789"` em ASCII com esses parâmetros é `0xF4`.

## Parte 2: fila persistente de resultados da jig

Uma jig de fábrica testa cada placa e precisa enviar o resultado para uma API. A rede cai. A energia cai. O resultado não pode sumir nem chegar fora de ordem.

Implemente uma fila de até 32 registros gravada em flash. Cada registro tem um número de série de 12 caracteres, um resultado (aprovado ou reprovado com código de falha de 1 byte) e um timestamp u32.

Operações mínimas:

- `init`: monta a fila a partir do conteúdo da flash. Chamar `init` de novo sobre a mesma flash simula um reboot.
- `push`: grava um registro novo.
- `peek`: devolve o mais antigo sem remover.
- `ack`: remove o mais antigo depois que a API confirmou o recebimento.

Requisitos:

- Se a escrita na flash falhar, a operação retorna erro e a fila continua exatamente como estava, na RAM e na flash.
- Um reboot entre quaisquer duas operações não perde registro confirmado na flash e não muda a ordem.
- Um registro só sai da fila depois de `ack` bem-sucedido.
- Decida o que acontece com a fila cheia e justifique.

Para testar, escreva uma flash fake em RAM que permita injetar falha de escrita e simular reboot. Os testes de falha são a parte mais importante desta seção.

## Entrega

Trate o repositório como algo que outra pessoa do time vai manter a partir de amanhã.

- Um repositório público no GitHub. Crie um repositório novo em vez de fazer fork deste, assim a sua solução não aparece na lista de forks para os outros candidatos.
- O histórico de commits faz parte da avaliação.
- O README precisa explicar como compilar e rodar os testes numa máquina Linux limpa, as decisões de API, o que ficou de fora e o que você faria com mais tempo.

## Como a gente trabalha

Não é checklist. É só para você saber onde vai cair.

- C++ no padrão mais novo, hoje C++26, sem exceções, RTTI nem alocação dinâmica. Bare-metal no EFR32 da Silicon Labs, ESP-IDF com FreeRTOS no ESP32 e Zephyr no Nordic.
- CMake com presets e Ninja, chamados por receitas do `just`. O CI roda as mesmas receitas que a gente roda na máquina.
- Testes no host com GoogleTest e gMock, compilados com ASan e UBSan.
- clang-format, clang-tidy e cppcheck barram o PR.
- GitHub Actions em todo repositório, num container com o toolchain fixado. Todo PR passa por revisão humana e do CodeRabbit.
- Claude Code todo dia. O código que entra é responsabilidade de quem abriu o PR.
