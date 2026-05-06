# Real-DGX-Spark-performances
On a _tous_ phantasmé le DGX Spark dès son annonce en mars 2025 lors de la conférence GTC. Il a fallu attendre plus de 6 mois pour mettre la main dessus.

Mettre en œuvre un cluster DGX Spark, c'est goûter à la fois à l'ivresse de manipuler 360 Go de VRAM ARM64 dans son propre garage et à l'amertume de découvrir que NVFP4 plante silencieusement sur GB10 pour des modèles entiers, sans qu'aucune documentation officielle ne daigne le mentionner. 
Chaque victoire — un Gemma3-27B bien servi, un Qwen3.5-MoE qui tient enfin la charge — se paie d'heures de débogage face à des incompatibilités matérielles que NVIDIA refuse de reconnaître publiquement, laissant la communauté reverse-engineer ce qui devrait être documenté. 
La mise en réseau RoCE 200 Gbps a englouti un budget considérable en ConnectX-7, switch MikroTik CRS804 et câbles QSFP-DD/QSFP56, pour finalement se heurter à des conflits entre bridges Linux et l'eSwitch matériel — là encore, aucun guide constructeur, juste des fils de discussion GitHub épars. Le support officiel se résume à des forums clairsemés, des tickets sans réponse et l'absence totale de feuille de route lisible pour les correctifs firmware, le support NVFP4, ou même l'écosystème logiciel ARM64. 

Au final, le DGX Spark reste une machine fascinante, mais son utilisateur doit accepter d'être simultanément ingénieur, intégrateur et bêta-testeur non rémunéré, livré à lui-même par un constructeur qui vend une vision sans assumer le service après-vente qui devrait l'accompagner.

Il y a un certain nombre de choses dont on ne parle pas. Prenez un vrai Nvidia DGX SPark. Il n'a pas une traitre vraie LED d'état pour savoir si on l'a lancé ou pas, heureusement l'ASUS GX10 dispose d'une LED, [mais l'ASUS GX10 n'exploitait initialement pas toute la bande passante PCIe de la seconde interface QSFP56](https://forums.developer.nvidia.com/t/asus-ascent-gx10-second-connectx-7-pcie-slot-wired-at-gen5-x2-instead-of-x4-50-bandwidth-loss-vs-dgx-spark-fe/367853), j'ai posté mon message d'escuses pour la gêne occasionnée, mais il convient de bien prêter attention aux firmwares, dont les mises à jour sont relativement fréquentes, et qui peuvent interférer.

En vrai, ça fonctionne, mais il ne faut pas imaginer un seul instant que c'est plug and play. Oui, ok, on peut suivre des tutos disponibles sur le site NVIDIA, mais, attention, pour travailler avec et utiliser des modèles récents ou sexy, il faut transpirer.

Je vous livre ici quelques performances sur single GPU.

# Catalogue vLLM — DGX GB10 — 2026-05-06


**Moteur unique** : tous les modèles servables tournent via [vLLM](https://github.com/vllm-project/vllm) 

## Tableau

| # | Nom | Repo HF / chemin | Archi | Quant | `max_len` | dtype | parser | think | Moteur | Perf avg tok/s | TTFT short (ms) | Startup | Statut |
|---|---|---|---|---|---:|---|---|:-:|---|---:|---:|---:|---|
|  1 | Gemma 4 26B MoE                | cyankiwi/gemma-4-26B-A4B-it-AWQ-4bit         | MoE A4B | AWQ   | 131 072 | half     | gemma4      | —   | vLLM cu130-nightly | 45.4 | 113 | 281 s | ✅ |
|  2 | Nemotron 3 Nano 30B (think)    | stelterlab/NVIDIA-Nemotron-3-Nano-30B-A3B-AWQ| MoE A3B | AWQ   | 262 144 | half     | hermes      | yes | vLLM cu130-nightly | 84.7 | 111 | 262 s | ✅ |
|  3 | Nemotron 3 Nano 30B (no-think) | stelterlab/NVIDIA-Nemotron-3-Nano-30B-A3B-AWQ| MoE A3B | AWQ   | 262 144 | half     | hermes      | no  | vLLM cu130-nightly | **94.2** | 111 | 251 s | ✅ top |
|  4 | Qwen3.5-35B-A3B (think)        | cyankiwi/Qwen3.5-35B-A3B-AWQ-4bit            | MoE A3B | AWQ   | 262 144 | half     | hermes      | yes | vLLM cu130-nightly | 38.1 | 112 | 326 s | ✅ |
|  5 | Qwen3.5-35B-A3B (no-think)     | cyankiwi/Qwen3.5-35B-A3B-AWQ-4bit            | MoE A3B | AWQ   | 262 144 | half     | hermes      | no  | vLLM cu130-nightly | 38.2 | 113 | 326 s | ✅ |
|  6 | Gemma 4 31B                    | cyankiwi/gemma-4-31B-it-AWQ-4bit             | dense   | AWQ   | 131 072 | half     | gemma4      | —   | vLLM cu130-nightly | 10.7 | 223 | 392 s | ✅ |
|  7 | Qwen3-Coder 30B (think)        | QuantTrio/Qwen3-Coder-30B-A3B-Instruct-AWQ   | MoE A3B | AWQ   | 262 144 | half     | hermes      | yes | vLLM cu130-nightly | 85.8 | 112 | 336 s | ✅ |
|  8 | Qwen3-Coder 30B (no-think)     | QuantTrio/Qwen3-Coder-30B-A3B-Instruct-AWQ   | MoE A3B | AWQ   | 262 144 | half     | hermes      | no  | vLLM cu130-nightly | 86.6 | 111 | 317 s | ✅ |
| 10 | Qwen3.6-35B-A3B NVFP4 (think)  | RedHatAI/Qwen3.6-35B-A3B-NVFP4               | MoE A3B | NVFP4 | 262 144 | half     | qwen3_coder | yes | vLLM cu130-nightly | 42.6 | 110 | 477 s | ✅ |
| 11 | Qwen3.6-35B-A3B NVFP4 (no-think)| RedHatAI/Qwen3.6-35B-A3B-NVFP4              | MoE A3B | NVFP4 | 262 144 | half     | qwen3_coder | no  | vLLM cu130-nightly | 42.9 | 113 | 397 s | ✅ |
| 12 | Gemma 4 31B NVFP4 turbo        | LilaRest/gemma-4-31B-it-NVFP4-turbo          | dense   | NVFP4 |  16 384 | bfloat16 | gemma4      | —   | vLLM cu130-nightly | 10.5 | 222 | 291 s | ✅ |
| 13 | Qwen3.6-27B NVFP4 (think)      | sakamakismile/Qwen3.6-27B-NVFP4              | dense   | NVFP4 | 262 144 | bfloat16 | qwen3_coder | yes | vLLM cu130-nightly | 12.2 | 165 | 422 s | ✅ |
| 14 | Qwen3.6-27B NVFP4 (no-think)   | sakamakismile/Qwen3.6-27B-NVFP4              | dense   | NVFP4 | 262 144 | bfloat16 | qwen3_coder | no  | vLLM cu130-nightly | 12.3 | 168 | 412 s | ✅ |
| 15 | Qwen3.6-35B-A3B AWQ (think)    | cyankiwi/Qwen3.6-35B-A3B-AWQ-4bit            | MoE A3B | AWQ   | 262 144 | half     | qwen3_coder | yes | vLLM cu130-nightly | 37.7 | 113 | 306 s | ✅ |
| 16 | Qwen3.6-35B-A3B AWQ (no-think) | cyankiwi/Qwen3.6-35B-A3B-AWQ-4bit            | MoE A3B | AWQ   | 262 144 | half     | qwen3_coder | no  | vLLM cu130-nightly | 37.7 | 112 | 326 s | ✅ |
| 17 | Qwen3.6-27B AWQ (think)        | cyankiwi/Qwen3.6-27B-AWQ-INT4                | dense   | AWQ   | 262 144 | half     | qwen3_coder | yes | vLLM cu130-nightly | 12.3 | 166 | 436 s | ✅ |
| 18 | Qwen3.6-27B AWQ (no-think)     | cyankiwi/Qwen3.6-27B-AWQ-INT4                | dense   | AWQ   | 262 144 | half     | qwen3_coder | no  | vLLM cu130-nightly | 12.4 | 166 | 437 s | ✅ |
| 19 | Nemotron 3 Nano 30B NVFP4 (think) | nvidia/NVIDIA-Nemotron-3-Nano-30B-A3B-NVFP4 | MoE A3B | NVFP4 | 262 144 | half     | hermes      | yes | vLLM cu130-nightly | 63.0 | 113 | 566 s | ✅ |
| 20 | Nemotron 3 Nano 30B NVFP4 (no-think)| nvidia/NVIDIA-Nemotron-3-Nano-30B-A3B-NVFP4 | MoE A3B | NVFP4 | 262 144 | half     | hermes      | no  | vLLM cu130-nightly | 65.7 | 112 | 276 s | ✅ |
| 21 | Nemotron 3 Nano Omni NVFP4     | nvidia/Nemotron-3-Nano-Omni-30B-A3B-Reasoning-NVFP4 | MoE A3B + audio/vision | NVFP4 | 131 072 | bfloat16 | hermes      | yes | vLLM cu130-nightly (KO) | — | — | — | ❌ KeyError |
| 22 | Nemotron 3 Nano Omni AWQ       | feanors/nemotron-3-nano-omni-30b-a3b-awq     | MoE A3B + audio/vision | AWQ   | 131 072 | bfloat16 | hermes      | yes | vLLM cu130-nightly (KO) | — | — | — | ❌ KeyError |
| 23 | Nemotron 3 Nano Omni FP8       | nvidia/Nemotron-3-Nano-Omni-30B-A3B-Reasoning-FP8 | MoE A3B + audio/vision | FP8   | 131 072 | auto     | hermes      | yes | vLLM cu130-nightly (KO) | — | — | — | ❌ KeyError |

## `extra_args` non triviaux

- **#10** Qwen3.6-35B NVFP4 think : `--reasoning-parser qwen3 --moe-backend marlin`
- **#11** Qwen3.6-35B NVFP4 no-think : `--moe-backend marlin`
- **#12** Gemma 4 31B turbo : `--quantization modelopt --kv-cache-dtype fp8 --trust-remote-code --enable-prefix-caching`
- **#13/#14** Qwen3.6-27B NVFP4 : `--quantization compressed-tensors --trust-remote-code --kv-cache-dtype fp8 --reasoning-parser qwen3`
- **#15** Qwen3.6-35B AWQ think : `--reasoning-parser qwen3`
- **#17** Qwen3.6-27B AWQ think : `--reasoning-parser qwen3`
- **#19/#20** Nemotron Nano NVFP4 : `--moe-backend marlin`
- **#21/#22/#23** Omni : `--trust-remote-code --limit-mm-per-prompt {"video":0,"image":0,"audio":0}`

## Légende

- **Archi** : `MoE A3B` = mixture-of-experts avec 3 B params actifs par token ; `MoE A4B` = 4 B actifs ; `dense` = activation totale du modèle.
- **Quant** : `AWQ` = AutoAWQ 4-bit ; `NVFP4` = NVIDIA 4-bit float ; `FP8` = NVIDIA 8-bit float (full).
- **Perf avg tok/s** : moyenne sur 3 prompts (short 100 tok / medium 400 tok / long 800 tok), génération pure post-prefill, temp=0.7.
- **TTFT short** : temps entre la requête et le 1er token streamé (`stream_options.include_usage=true`).
- **Startup** : durée entre lancement docker et 1ère réponse `/v1/chat/completions`.
- **Statut** : ✅ servable / ❌ KO. Pour les Omni : `KeyError: '0.weight'` à `vllm/model_executor/models/nano_nemotron_vl.py:1526` — bug upstream vLLM (cu130-nightly et cu129-nightly), cf. mémoire `feedback_vllm_omni_keyerror.md`.

## Particularités à connaître

| ID(s) | Note |
|---|---|
| **#2 / #3** Nemotron Nano AWQ | Modèle reasoning par construction. Le `start.sh` patche le chat template pour mettre `enable_thinking=false` en mode no-think (#3), mais le modèle pose quand même un `</think>` orphelin → **avec `--tool-call-parser hermes` seul, le raisonnement fuit dans `content`**. **Fix : ajouter `--reasoning-parser deepseek_r1`** au lancement (ce parser accepte le `</think>` sans balise ouvrante). À mettre dans `extra_args` de `models.json` si servi en prod. Qualité multilingue **médiocre en FR** (mélange spontané FR/EN). |
| **#10 / #11** Qwen3.6-35B-A3B NVFP4 | **Excellente qualité FR** en mode think (#10). Reasoning séparé propre, `content` 100 % FR. Compter ~300-1000 tokens de raisonnement ⇒ **prévoir `max_tokens` ≥ 2 000** côté client pour ne pas tronquer la réponse finale. `--moe-backend marlin` est obligatoire (kernel NVFP4). |
| **#12** Gemma 4 31B NVFP4 turbo | `max_model_len = 16 384` seulement (vs 131 K natif Gemma 4). Volontairement bridé : `--kv-cache-dtype fp8 + --enable-prefix-caching` font exploser la KV cache à long contexte. Pour un context window plus large, baisser `--gpu-memory-utilization` ou désactiver prefix caching. |
| **#13 / #14** Qwen3.6-27B NVFP4 | `--quantization compressed-tensors` + `--kv-cache-dtype fp8`. Performance **memory-bound** (~12 tok/s), comme tout dense 27B. |
| **#15-#18** Qwen3.6 AWQ | Quantization récente (avril 2026), aussi rapide ou supérieure aux NVFP4 sur GB10 pour Qwen3.6 dense ; sur le 35B MoE, NVFP4 +13 % en débit. |
| **#19 / #20** Nemotron Nano NVFP4 (officiel NVIDIA) | **Anomalie GB10 : ~30 % plus lent que l'AWQ** (~63 tok/s vs ~85-94 tok/s) même avec `--moe-backend marlin`. Cause exacte non identifiée (régression NVFP4 sur archi `NemotronHForCausalLM` dans vLLM cu130-nightly). **Préférer l'AWQ #2/#3 pour ce modèle**. |
| **#21 / #22 / #23** Nemotron Omni (NVFP4 / AWQ / FP8) | ❌ **Tous KO** sur cu130-nightly (testé 2026-05-06) et cu129-nightly. Erreur : `KeyError: '0.weight'` à `vllm/model_executor/models/nano_nemotron_vl.py:1526` lors du `load_weights` de l'adapter multimodal. Bug upstream vLLM, pas d'issue ouverte. Workaround : aucun connu aujourd'hui. |
| Tous les Qwen3.6 think (#10, #13, #15, #17) | `--reasoning-parser qwen3` actif, le `reasoning` arrive bien dans son champ propre. Latence dominée par la phase de pensée — prévoir des `max_tokens` larges (2 000+) ou switcher sur la variante no-think. |
