# MPEG-DASH Adaptive Streaming com Q-Learning

Algoritmo de **Adaptive Bitrate (ABR)** para um cliente **MPEG-DASH** em que a qualidade de cada segmento de vídeo é escolhida por um agente de **aprendizado por reforço (Q-Learning)** com política de seleção **Softmax**, em vez das heurísticas fixas usadas pelos clientes HAS tradicionais.

Projeto final da disciplina **Redes de Computadores (CIC0124)** — Departamento de Ciência da Computação, Universidade de Brasília (UnB), 2025.

[![Python](https://img.shields.io/badge/Python-3.x-3776AB?logo=python&logoColor=white)](https://www.python.org/)
[![NumPy](https://img.shields.io/badge/NumPy-013243?logo=numpy&logoColor=white)](https://numpy.org/)
[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)

---

## 🎥 Apresentação em vídeo

| Vídeo | Link |
|---|---|
| Explicação completa do trabalho | [YouTube](https://www.youtube.com/watch?v=iwGN1svwxLg&t=2239s) |
| Versão resumida | [YouTube](https://www.youtube.com/watch?v=jfvYykvi8lo&t=1165s) |

📄 Relatório técnico completo: [`docs/relatorio.pdf`](docs/relatorio.pdf)

---

## Sumário

- [Visão geral](#visão-geral)
- [Como funciona](#como-funciona)
- [Resultados](#resultados)
- [Estrutura do repositório](#estrutura-do-repositório)
- [Como executar](#como-executar)
- [Referências](#referências)

---

## Visão geral

O streaming adaptativo via HTTP (DASH) divide o vídeo em segmentos codificados em várias qualidades; o cliente decide, segmento a segmento, qual qualidade baixar. Heurísticas determinísticas costumam ser ajustadas para cenários de rede específicos e perdem desempenho fora deles.

Aqui, o cliente **aprende** a política de seleção observando o estado do player e recebendo recompensas que equilibram qualidade, estabilidade e saúde do buffer.

- **Vídeo:** *Big Buck Bunny* — 596 s, apenas vídeo, **20 representações** de 46.980 bps a 4.726.737 bps
- **Ambiente:** framework **pyDash** (módulo R2A), com largura de banda variando segundo uma distribuição exponencial
- **Hiperparâmetros:** taxa de aprendizado **α = 0.3**, fator de desconto **γ = 0.95**, temperatura Softmax **τ = 1.0** no treino e **τ = 0.5** na execução

## Como funciona

### Espaço de estados

| Elemento | Faixa |
|---|---|
| Preenchimento do buffer | `[0 ; Bmax]` s |
| Variação do buffer | `[-Bmax ; Bmax]` s |
| Nível de qualidade atual | `[1 ; N]` |
| Largura de banda (discretizada nas qualidades disponíveis) | `[0 ; BWmax]` bps |
| Comprimento da oscilação | `[0 ; 60]` segmentos |
| Profundidade da oscilação | `[0 ; N-1]` |

A **ação** é a escolha de uma das N = 20 qualidades. A Q-table é um dicionário `estado → array de 20 valores Q`.

### Ciclo de aprendizado

1. **Pré-treino offline** — na requisição do primeiro segmento, o agente simula episódios com estados aleatórios para popular a Q-table antes de tomar decisões reais.
2. **`handle_segment_size_request`** — observa o estado real (via `whiteboard` do pyDash), calcula probabilidades com Softmax e escolhe a qualidade.
3. **`handle_segment_size_response`** — mede a vazão, calcula a recompensa, determina o próximo estado e atualiza a Q-table:

$$Q(s,a) \leftarrow Q(s,a) + \alpha \left[ r + \gamma \max_{a'} Q(s',a') - Q(s,a) \right]$$

### Seleção Softmax

$$P(a) = \frac{e^{Q(s,a)/\tau}}{\sum_{j} e^{Q(s,a_j)/\tau}}$$

Favorece ações com maior valor Q sem eliminar a exploração.

### Função de recompensa

Baseada em [Claeys et al.], combina quatro componentes, com peso maior para a estabilidade do buffer:

$$R = 2\,R_{qualidade} + 1\,R_{oscilação} + 4\,R_{buffer} + 3\,R_{variação\ do\ buffer}$$

## Resultados

Execução completa do vídeo (~10 min) no pyDash:

| Métrica | Valor |
|---|---|
| Número de pausas | 7 |
| Duração média das pausas | 3,14 s (≈ 21 s no total) |
| Qualidade média (QI) | 9,3 (desvio padrão 5,8) |
| Distância média entre QIs consecutivos | 6,48 |

O agente reduz a qualidade quando percebe o buffer esvaziando e a aumenta quando o buffer está confortável, mantendo poucas interrupções. A alta variância de qualidade indica oscilações frequentes — o principal ponto de melhoria.


</details>

## Estrutura do repositório

```
mpeg-dash-qlearning-abr/
├── src/
│   └── r2a/
│       └── r2aqlearning.py      # Algoritmo ABR (classe R2AQlearning)
├── docs/
│   ├── relatorio.pdf            # Relatório técnico
│   └── figures/                 # Gráficos de resultados
├── requirements.txt
├── LICENSE
└── README.md
```

## Como executar

O algoritmo é um módulo **R2A** e roda dentro do framework **pyDash**, usado na disciplina.

1. Clone o pyDash e instale as dependências:
   ```bash
   pip install -r requirements.txt
   ```
2. Copie `src/r2a/r2aqlearning.py` para a pasta `r2a/` do pyDash.
3. No `dash_client.json`, defina o algoritmo R2A como `R2AQlearning` e aponte o MPD para o dataset:
   `http://45.171.101.167/DASHDataset/` (Big Buck Bunny, segmentos de 1 s).
4. Execute o cliente:
   ```bash
   python main.py
   ```

> **Observação:** o código atual usa 100.000 episódios de pré-treino (o relatório menciona 200.000). Esse valor pode ser ajustado no laço de treino em `handle_segment_size_request`.

## Possíveis melhorias

- Reduzir oscilações aumentando o peso de `R_oscilação` ou usando decaimento de temperatura no Softmax
- Discretizar melhor o estado para diminuir o tamanho da Q-table
- Substituir a Q-table por uma rede neural (Deep Q-Learning)
- Comparar com heurísticas clássicas (throughput-based, BOLA) no mesmo cenário

## Referências

- Claeys, M. et al. *Design of a Q-learning-based client quality selection algorithm for HTTP adaptive video streaming.* Ghent University – iMinds / Alcatel-Lucent Bell Labs.
- Awan, A. A. (2024). *An Introduction to Q-Learning: A Tutorial for Beginners.* DataCamp.

## Autor

**Felipe Perone** — Engenharia Mecatrônica, Universidade de Brasília

## Licença

Distribuído sob a licença MIT. Veja [`LICENSE`](LICENSE).
