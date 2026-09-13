# Phase 15 — Workflow Git OSS

> **Prérequis :** Phases [1](./phase-1-what-is-openhikmah.md)–[14](./phase-14-engineering-culture.md) — setup local (Phase 10), tests (Phase 13), conventions PR (Phase 14)  
> **Balises de preuve :** **FACT** · **INFERENCE** · **UNKNOWN**

La Phase 14 a couvert **comment** les changements sont reviewés. La Phase 15 couvre **où** vos commits vivent dans git — fork, upstream, branches, push, et ouvrir une PR vers `OpenHikmah/openhikmah-web`.

**Cette phase est surtout en lecture seule** — vous vérifiez vos remotes et parcourez le workflow sur papier. Phase 16 choisit votre première cible de contribution.

---

## Le modèle mental trois dépôts

```mermaid
flowchart LR
  UP["upstream\nOpenHikmah/openhikmah-web\n(canonical)"]
  FK["origin\nyour fork\nradhtkamal/openhikmah-web"]
  LOC["local clone\nyour machine"]

  UP -->|"fetch / merge"| LOC
  LOC -->|"push feature branch"| FK
  FK -->|"Pull Request"| UP
```

| Remote | URL (votre machine) | Rôle |
| --- | --- | --- |
| **`upstream`** | `https://github.com/OpenHikmah/openhikmah-web.git` | Projet canonique — **lisez** les mises à jour ici ; **ne push jamais** direct sauf si vous êtes maintainer |
| **`origin`** | `https://github.com/radhtkamal/openhikmah-web.git` | **Votre fork** — push vos branches feature ici |
| **local** | `/Users/eihdar/Documents/ChatGPT/openhikmah-web` | Où vous branchez, committez, et lancez les hooks |

**FACT:** Inspecté sur votre clone (Sep 2026) :

```text
origin   → https://github.com/radhtkamal/openhikmah-web.git
upstream → https://github.com/OpenHikmah/openhikmah-web.git
```

**FACT:** GitHub confirme `radhtkamal/openhikmah-web` est un **fork** de `OpenHikmah/openhikmah-web`.

**FACT:** Votre `main` est au commit `a472416` et matche **à la fois** `origin/main` et `upstream/main` (0 commits behind/ahead upstream après fetch).

**INFERENCE:** GitHub Desktop a déjà configuré les remotes correctement — vous pouvez sauter le setup « how to add upstream » et vous concentrer sur la boucle continue.

---

## Nommage de branche et où vivent les branches

**FACT** (Phase 14) : les branches utilisent `feat/`, `fix/`, `chore/`, ou `docs/` + description kebab-case.

**Règle :** Créez les branches feature depuis **`main` fraîche**, pas depuis du travail stale :

```text
upstream/main  ──merge──►  local main  ──branch──►  fix/my-contribution
                                              │
                                              └── push ──► origin/fix/my-contribution
```

**INFERENCE:** Gardez **`main` sur votre fork alignée avec `upstream/main`** — utilisez-la comme branche de sync, pas branche de dev longue durée. Tout le travail se fait sur des branches feature nommées.

**FACT:** `scripts/precommit-checks.mjs` **bloque les commits directement sur `main`** — même en local vous devez utiliser une branche feature.

---

## La boucle contributeur (chaque PR)

### 0. Avant de commencer — sync

```bash
git checkout main
git fetch upstream
git merge upstream/main          # or: git rebase upstream/main
git push origin main             # keep fork main current
```

**Pourquoi :** Réduit les conflits merge et assure que CI tourne contre le code latest.

**Votre statut maintenant :** Déjà synced à `a472416` — cette étape serait un no-op aujourd'hui.

---

### 1. Créer une branche feature

```bash
git checkout -b fix/short-description
```

Exemples alignés avec l'historique du dépôt :

```text
fix/search-dialog-focus-trap
docs/onboarding-typo
chore/readme-docker-note
```

Choisissez **`fix/`** vs **`feat/`** vs **`chore/`** honnêtement — auto-triage mappe les préfixes titre aux labels (`fix`→`bug`, `feat`→`enhancement`).

---

### 2. Faire changements + valider

Minimum avant commit (Phase 13 + 14) :

```bash
bun run format:check
bun run lint
bun run typecheck
bun run test:ci
```

Avant **push** (hook pre-push) :

```bash
docker info                      # must succeed
# hook runs: bun run test:integration
```

**FACT:** Sauter `--no-verify` est déconseillé sauf instruction explicite maintainer.

---

### 3. Commit

Conventional commit avec scope :

```bash
git add <files>
git commit -m "$(cat <<'EOF'
fix(canvas): prevent duplicate edge on rapid expand

EOF
)"
```

Si un fichier entier neuf ou ~30+ lignes étaient générés IA avec peu d'édition, ajoutez le trailer selon `AGENTS.md` :

```text
Generated-By: Cursor
```

**Pas de `Co-Authored-By`.**

---

### 4. Push vers **votre fork** (`origin`)

```bash
git push -u origin fix/short-description
```

**FACT:** Le premier push d'une branche a besoin de `-u` pour que les pushes futurs puissent être un simple `git push`.

**INFERENCE:** Vous push vers **`origin`**, jamais vers **`upstream`**, en tant que contributeur externe.

---

### 5. Ouvrir PR vers **OpenHikmah/main**

Dépôt cible : **`OpenHikmah/openhikmah-web`**, branche base **`main`**, branche compare **`radhtkamal:fix/short-description`**.

**CLI** (quand prêt) :

```bash
gh pr create \
  --repo OpenHikmah/openhikmah-web \
  --base main \
  --head radhtkamal:fix/short-description \
  --title "fix(canvas): prevent duplicate edge on rapid expand" \
  --body "$(cat <<'EOF'
## Summary

- …

## Type of change

- [x] Bug fix

## AI / Theological changes

- [x] No AI or theological changes in this PR

## Testing

- [x] `bun run test:ci` passes locally
- [x] `bun run typecheck` passes locally
- [x] `bun run lint` passes locally
- [x] `bun run format:check` passes locally
- [x] New tests added for new functionality

## Checklist

- [x] Branch is up to date with `main`
- [x] Conventional commits
- [x] No secrets or real API keys in the diff
EOF
)"
```

**UI GitHub :** Fork → « Contribute » → « Open pull request » → assurez-vous que le repo base est **OpenHikmah/openhikmah-web**, pas votre fork.

**INFERENCE:** Les PR depuis forks ont `GITHUB_TOKEN` read-only pour certains commentaires bot — CI tourne quand même entièrement ; commentaires bundle-size utilisent un workflow follow-up (Phase 14).

---

### 6. Pendant review — rester à jour

Quand `upstream/main` avance pendant que votre PR est ouverte :

```bash
git checkout fix/short-description
git fetch upstream
git merge upstream/main            # or rebase if you prefer linear history
# resolve conflicts if any
bun run test:ci                    # re-verify
git push origin fix/short-description
```

**INFERENCE:** Les maintainers préfèrent souvent **merge commits ou merge-upstream** plutôt que force-push pour premiers contributeurs — demandez si incertain. Évitez `git push --force` sauf si review demande explicitement rebase + force.

Répondez à la review en **nouveaux commits** sur la même branche — pattern de PR #598 : citez feedback, fixez, push encore.

---

### 7. Après merge — nettoyer

```bash
git checkout main
git fetch upstream
git merge upstream/main
git push origin main

git branch -d fix/short-description
git push origin --delete fix/short-description   # optional
```

**INFERENCE:** Supprimer branches mergées garde le fork propre ; GitHub peut auto-supprimer head branches si activé dans settings fork.

---

## GitHub Desktop vs CLI

Vous avez cloné via **GitHub Desktop** — mappings équivalents :

| Intention | GitHub Desktop | CLI |
| --- | --- | --- |
| Sync depuis canonique | Fetch `upstream`, merge into `main` | `git fetch upstream && git merge upstream/main` |
| Nouvelle branche | Branch → New branch | `git checkout -b fix/…` |
| Commit | Commit panel | `git commit` |
| Push | Push origin | `git push -u origin branch` |
| Ouvrir PR | Branch → Create pull request | `gh pr create --repo OpenHikmah/openhikmah-web …` |

**FACT:** Votre CLI `gh` est authentifié comme **`radhwana`** alors que le remote fork est **`radhtkamal`** — les deux peuvent coexister si `radhwana` a accès push à `radhtkamal/openhikmah-web`. Si `gh pr create` échoue avec erreurs permission, utilisez `--head radhtkamal:branch` explicitement ou ouvrez la PR dans le navigateur.

**UNKNOWN:** Si « Create PR » de GitHub Desktop default vers base `OpenHikmah/main` — **vérifiez toujours** le repo base avant soumission.

---

## Ce qu'il ne faut **pas** faire

| Erreur | Pourquoi ça fait mal |
| --- | --- |
| PR depuis `main` avec commits mélangés non liés | Dur à review ; pre-commit bloquait commits sur main anyway |
| PR base = seulement `main` de votre fork | Ne contribue pas upstream — doit cibler **OpenHikmah/openhikmah-web** |
| Push vers `upstream` | Permission denied (sauf maintainer) |
| `--no-verify` sur commit/push | Saute hooks ; CI peut quand même échouer ; viole normes dépôt |
| Force-push `main` | Jamais nécessaire pour flux OSS normal |
| Commit `.env.local` ou clés API | gitleaks + rejet review |
| Grosse PR touchant auth + IA + UI | CODEOWNERS + review théologique — split (Phase 14) |

---

## Cas spécial : vos docs onboarding

**FACT:** La ligne 46 de `.gitignore` ignore tout le répertoire `docs/` :

```gitignore
docs/
```

Tous les fichiers sous `docs/onboarding/` (Phases 1–15) sont **local-only** — ils n'apparaissent pas dans `git status` et **ne peuvent pas être PR'd as-is**.

| Si vous voulez… | Approche |
| --- | --- |
| Garder docs perso | Ne rien faire — setup actuel OK pour apprendre |
| Contribuer onboarding upstream | Ouvrir une **issue d'abord** proposant docs in-repo ; peut nécessiter changement `.gitignore` ou déplacer docs vers chemin non ignoré (décision maintainer) |
| Pratique première PR | Choisir cible **code** de Phase 16 — pas la série onboarding ignorée |

**INFERENCE:** Traitez onboarding comme carnet d'étude privé jusqu'à accord maintainer sur politique doc upstream.

---

## Checklist hygiène fork

Lancez cet exercice de vérification maintenant (lecture seule sauf fetch) :

```bash
# 1. Remotes
git remote -v

# 2. Sync state
git fetch upstream
git status -sb
git rev-list --count main..upstream/main    # should be 0 when current

# 3. Fork relationship
gh repo view radhtkamal/openhikmah-web --json isFork,parent

# 4. Hooks present
test -x .husky/pre-commit && test -x .husky/pre-push && echo "hooks OK"

# 5. Docker (for future push)
docker info >/dev/null && echo "docker OK" || echo "docker NOT running — fix before push"
```

**Attendu sur votre machine aujourd'hui :**

| Check | Attendu |
| --- | --- |
| `origin` | `radhtkamal/openhikmah-web` |
| `upstream` | `OpenHikmah/openhikmah-web` |
| Behind upstream | `0` |
| Working tree | clean |
| `docs/onboarding/` | present locally, **ignored by git** |

---

## Dry-run : branche sans commit

Répétition mentale optionnelle — crée une branche, puis la supprime :

```bash
git checkout main
git pull upstream main          # no-op if current
git checkout -b chore/phase15-dry-run
# …would edit files here…
git checkout main
git branch -D chore/phase15-dry-run
```

Pas de push, pas de PR — confirme le workflow branche sans bruit sur votre fork.

---

## CI tourne aussi sur votre fork

**FACT:** Push une branche vers `origin` déclenche GitHub Actions sur **`radhtkamal/openhikmah-web`** (même fichier workflow qu'upstream).

**INFERENCE:** Vous pouvez voir CI vert sur votre fork **avant** d'ouvrir la PR upstream — utile pour premières contributions.

**INFERENCE:** Ouvrez la PR seulement quand CI fork passe — économise du temps maintainer.

---

## MUST UNDERSTAND NOW

1. **`upstream`** = source lecture canonique ; **`origin`** = votre fork pour pushes.
2. **Cible PR** = `OpenHikmah/openhikmah-web` **`main`**, head = `radhtkamal:<branch>`.
3. **Ne jamais commit sur `main` local** — branches feature seulement (imposé par hook).
4. **Boucle sync :** `fetch upstream` → update `main` local → branche → push `origin` → PR → après merge, sync encore.
5. **pre-push a besoin Docker** — tests intégration tournent avant push réussi.
6. **`docs/onboarding/` est gitignored** — pas partie de votre première PR sauf changement politique.
7. Votre fork est **déjà configuré et synced** — vous êtes prêt pour une branche feature quand Phase 16 choisit une cible.

---

## USEFUL LATER

- `gh pr checks` — surveillez CI sur votre PR
- `gh pr view --web` — commentaires review
- Activez « Automatically delete head branches » sur votre fork
- `git config pull.rebase false` vs `true` — choisissez un style rebase/merge et restez cohérent

---

## IGNORE FOR NOW

- Contribuer directement à `OpenHikmah` sans fork (maintainer-only)
- Git worktrees / stacked PRs — pas utilisés dans l'historique de ce dépôt
- Signing commits — pas requis sauf changement settings repo
- Publier docs onboarding — décision séparée des contributions code

---

## Questions checkpoint Phase 15

1. Quelle est la différence entre `origin` et `upstream` sur votre clone ?
2. Quel repo GitHub doit être la **base** à l'ouverture d'une PR de contribution ?
3. Pourquoi ne pouvez-vous pas PR `docs/onboarding/phase-14-engineering-culture.md` aujourd'hui ?
4. Qu'est-ce qui tourne sur `git push` qui ne tourne **pas** sur `git commit` ?
5. Après merge de votre PR, quelles trois commandes resync le `main` de votre fork ?

---

**Suivant :** [Phase 16 — Carte surface contribution & readiness](./phase-16-contribution-readiness.md) — cibles première PR classées, risques, et checklist readiness honnête.

**Votre action :** Lancez la **Checklist hygiène fork**, puis répondez **« continuer vers la phase 16 »** — ou demandez un walkthrough pour ouvrir une dry-run PR avec commit vide (toujours lecture seule si vous supprimez la branche avant push).

> **Version complète (français B2+) :** [Phase 15](../onboarding-fr/phase-15-oss-git-workflow.md)
