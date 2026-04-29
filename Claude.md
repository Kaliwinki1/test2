# ProtoPirate — Notes de projet

## Objectif
Activer la fonctionnalité TX (émulation/retransmission) dans ProtoPirate pour Flipper Zero sous firmware **Momentum**.

---

## Repo utilisé
**RocketGod-git/ProtoPirate**
https://github.com/RocketGod-git/ProtoPirate

> Note : Le repo officiel désactive volontairement le TX par défaut pour éviter la désynchronisation accidentelle des clés de voiture. Il faut patcher manuellement.

---

## Modifications appliquées

### 1. `defines.h` — Activer la feature TX
```c
// Avant :
// #define ENABLE_EMULATE_FEATURE

// Après :
#define ENABLE_EMULATE_FEATURE
```

### 2. `application.fam` — Permissions et stack size
```python
# Avant :
requires=["gui"],
stack_size=2 * 1024,

# Après :
requires=["gui","subghz"],
stack_size=8 * 1024,
```

**Pourquoi `stack_size=8 * 1024` ?**
Le Flipper Zero a 190KB de heap total. Avec toutes les features TX activées, il ne reste que ~15KB libre avec le stack par défaut de 2KB. Augmenter à 8KB évite le crash "Out of Memory" au lancement. Passer à `10 * 1024` si le crash persiste.

**Pourquoi `subghz` dans `requires` ?**
Nécessaire pour que le firmware Momentum autorise l'accès au stack Sub-GHz en mode TX.

---

## Compilation

### Pour Momentum (firmware actuel)
```bash
ufbt update --index-url https://get.momentum-fw.com/directory.json --channel dev
ufbt
```

### Pour Unleashed
```bash
ufbt update --index-url https://up.unleashedflip.com/directory.json --channel dev
ufbt
```

> ⚠️ Le canal de compilation **doit** correspondre au firmware installé sur le Flipper, sinon crash au démarrage.

Le `.fap` compilé se trouve dans `./dist/`.

---

### 3. `protopirate_app.c` — Compatibilité Momentum 12
```c
// Avant :
app->view_dispatcher = view_dispatcher_alloc();
#if defined(FW_ORIGIN_RM)
    view_dispatcher_enable_queue(app->view_dispatcher);
#endif

// Après :
app->view_dispatcher = view_dispatcher_alloc();
view_dispatcher_enable_queue(app->view_dispatcher);
```
`FW_ORIGIN_RM` est le flag pour Roguemaster (ancien nom de Momentum). Dans Momentum 12, `view_dispatcher_enable_queue` est requis mais le flag n'est pas défini automatiquement par `ufbt`, ce qui cause un crash au démarrage.

---

## Problèmes rencontrés

| Problème | Cause | Solution |
|---|---|---|
| "Not enough RAM" au lancement | `stack_size` trop petit (2KB) | Augmenter à `8 * 1024` |
| TX ne fonctionne pas | `ENABLE_EMULATE_FEATURE` commenté | Décommenter dans `defines.h` |
| Crash sur Receive | Même cause que RAM | `stack_size=8 * 1024` ou `10 * 1024` |
| App incompatible | SDK mismatch firmware/compilation | Recompiler avec le bon `--index-url` |

---

## Références Reddit
- [Script bash piratebuild.sh (t4c)](https://www.reddit.com/r/flipperhacks/comments/1rxa3cx/simple_linux_bashscript_to_enable_emulation_in/)
- [Protopirate Crash Out of Memory](https://www.reddit.com/r/flipperhacks/comments/1qmaq67/protopirate_crash_out_of_memory/)
- [Emulating captured signals with Protopirate](https://www.reddit.com/r/flipperhacks/comments/1qivtvi/emulating_captured_signals_with_protopirate/)

---

## Analyse des blocages TX cachés

RocketGod a ajouté un double verrou sur les encodeurs TX. Sans `ENABLE_EMULATE_FEATURE`, ces 8 protocoles ont leur encodeur mis à `NULL` (TX silencieusement désactivé) :

| Protocole | Encodeur bloqué sans define |
|---|---|
| `ford_v0.c` | `subghz_protocol_ford_v0_encoder` → NULL |
| `ford_v1.c` | `subghz_protocol_ford_v1_encoder` → NULL |
| `honda_static.c` | `subghz_protocol_honda_static_encoder` → NULL |
| `kia_v6.c` | `kia_protocol_v6_encoder` → NULL |
| `kia_v7.c` | `kia_protocol_v7_encoder` → NULL |
| `mazda_v0.c` | `subghz_protocol_mazda_v0_encoder` → NULL |
| `psa.c` | `subghz_protocol_psa_encoder` → NULL |
| `vag.c` | `subghz_protocol_vag_encoder` → NULL |

Ces protocoles compilent avec `#ifdef ENABLE_EMULATE_FEATURE ... #else .alloc = NULL #endif`. Décommenter le `#define` dans `defines.h` suffit pour les débloquer tous.

Protocoles **toujours actifs** même sans le define : `fiat_v0`, `fiat_v1`, `kia_v0` à `v5`, `mitsubishi_v0`, `porsche_touareg`, `scher_khan`, `star_line`, `subaru`.

---

## Contexte mémoire Flipper Zero
```
Heap total : 190 000 bytes
Heap libre avec toutes features ON : ~14 944 bytes  ← trop juste
Heap libre sans SUB_DECODE et TIMING_TUNER : ~28 192 bytes  ← OK
```
`ENABLE_TIMING_TUNER_SCENE` et `ENABLE_SUB_DECODE_SCENE` restent **commentés** dans `defines.h` pour économiser la RAM.
