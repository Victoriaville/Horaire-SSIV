# Restauration complète de Telegram (Horaire SSIV)

Ce document décrit, étape par étape, comment remettre en fonction toutes les fonctions Telegram qui ont été retirées de l'application en mars 2026. Il a été rédigé à partir de l'analyse de l'historique Git. Rien n'a été restauré au moment de la rédaction : l'application tourne actuellement sans Telegram (notifications dans l'application et par courriel seulement).

## 1. Références Git

| Élément | Valeur |
| --- | --- |
| Dernière version complète avec Telegram | commit `3833283` (16 mars 2026, fusion PR #70) |
| Commit principal de suppression des fichiers | `ffb98dc` (12 fichiers supprimés) |
| Autres commits de nettoyage | `994f919` (notifications.tsx, envoi manuel, assignation), `80162e3` (middleware, formulaire d'envoi, historique, script webhook), `e33920a` (onglet remplacements assignés), `77d4a64` (script de nettoyage de la base) |
| Script SQL qui a supprimé les données | `scripts/024-remove-telegram-from-database.sql` |

Règle d'or : ne jamais faire un `git revert` global. Cela annulerait aussi les corrections faites depuis (calendrier, remplacements, journal d'activités, mot de passe forcé, vérification des heures consécutives). On restaure uniquement les fichiers Telegram, un groupe à la fois.

## 2. À quoi sert le middleware

Le fichier `middleware.ts` s'exécute avant chaque page ou route de l'application. Il ne sert pas seulement à Telegram. Ses rôles actuels :

- Rediriger `/` vers `/dashboard` (connecté) ou `/login` (non connecté).
- Bloquer l'accès aux pages protégées si l'utilisateur n'est pas connecté (redirection vers `/login`).
- Empêcher un utilisateur connecté de retourner sur `/login` ou `/register`.
- Forcer la page de changement de mot de passe (`/dashboard/settings/password?reason=admin_reset`) quand `password_force_reset` est vrai.

Rôle lié à Telegram (retiré en mars 2026) : le webhook `/api/telegram/webhook` est appelé directement par les serveurs de Telegram, sans cookie de connexion. Sans exception dans le middleware, ces appels seraient redirigés vers `/login` et les boutons de confirmation de Telegram cesseraient de fonctionner. C'était la seule partie Telegram du middleware.

## 3. Ce qui doit être restauré

### 3.1 Fichiers supprimés à recréer tels quels (13)

À récupérer depuis `3833283` :

1. `lib/telegram.ts` : `sendTelegramMessage`, `getBotInfo`, `setTelegramWebhook`.
2. `app/actions/telegram.ts` : `generateTelegramLink`, `disconnectTelegram`, `checkTelegramStatus`, `setupTelegramWebhook`, `getTelegramWebhookInfo`.
3. `app/actions/telegram-status.ts` : `getTelegramConnectionStatus`, `checkUserTelegramStatus`, `toggleTelegramRequirement`.
4. `app/api/telegram/webhook/route.ts` : réception des messages Telegram (liaison par `/start CODE`, bouton de confirmation `confirm_replacement_<id>`).
5. `app/api/telegram/setup-webhook/route.ts` : enregistrement du webhook auprès de Telegram.
6. `components/telegram-connection-modal.tsx`
7. `components/telegram-connection-banner.tsx`
8. `components/telegram-status-table.tsx`
9. `components/telegram-alert-widget.tsx`
10. `components/telegram-webhook-setup.tsx`
11. `app/dashboard/settings/telegram-status/page.tsx`
12. `app/dashboard/settings/telegram-webhook/page.tsx`
13. `scripts/setup-telegram-webhook.ts`

### 3.2 Fichiers modifiés sans changement depuis le nettoyage (8) : restauration directe

Avant de restaurer chacun, vérifier qu'aucun autre changement n'a été fait depuis :

```
git log --oneline 3833283..HEAD -- <fichier>
git diff 3833283 HEAD -- <fichier>
```

Si le diff ne contient que du retrait de Telegram, on peut restaurer avec `git checkout 3833283 -- <fichier>`.

1. `app/actions/notifications.tsx` : envoi Telegram dans `createNotification` (environ 360 lignes), avec jetons de confirmation, nouvelles tentatives et suivi des canaux.
2. `app/actions/send-manual-notification.ts` : canal Telegram dans l'envoi manuel.
3. `app/actions/send-assignment-notification.tsx` : lecture de `enable_telegram` et `telegram_chat_id`.
4. `components/notification-preferences-form.tsx` : bouton de connexion et de déconnexion Telegram, interrupteur de canal.
5. `components/send-notification-form.tsx` : état Telegram par destinataire et statistiques d'envoi.
6. `app/dashboard/page.tsx` : widget `TelegramAlertWidget`.
7. `app/dashboard/settings/page.tsx` : cartes « Connexions Telegram » et « Configuration Telegram ».
8. `app/dashboard/settings/notification-history/[id]/page.tsx` : libellé du canal « Telegram ».

`app/dashboard/settings/send-notification/page.tsx` mentionne déjà « (app, Telegram) » dans son texte : rien à faire.

### 3.3 Fichiers à fusionner à la main (3) : NE PAS écraser

Ces trois fichiers ont reçu des changements importants depuis le nettoyage. Les remplacer par l'ancienne version ferait perdre ces changements.

**a) `middleware.ts`** (version actuelle : cookie chiffré JWE, vérification `password_force_reset`)

Ajouter seulement :

- Au tout début de la fonction `middleware`, avant tout autre test :
  ```ts
  if (pathname === "/api/telegram/webhook") {
    return NextResponse.next()
  }
  ```
- Dans `config.matcher`, ajouter l'exclusion du webhook :
  ```ts
  matcher: ["/((?!_next/static|_next/image|favicon.ico|api/telegram/webhook).*)"],
  ```

**b) `app/dashboard/layout.tsx`** (version actuelle : badges d'erreurs de notification, navigation mobile)

Voir l'ancienne version avec `git show 3833283:app/dashboard/layout.tsx` et réinsérer seulement :

- Les imports :
  ```ts
  import { TelegramConnectionBanner } from "@/components/telegram-connection-banner"
  import { TelegramConnectionModal } from "@/components/telegram-connection-modal"
  import { checkUserTelegramStatus } from "@/app/actions/telegram-status"
  ```
- Après la lecture de la session (`const user = await getSession()`) :
  ```ts
  const telegramStatus = user.is_admin ? { isConnected: true } : await checkUserTelegramStatus(user.id)
  ```
- La bannière, près du haut du JSX retourné (ancienne ligne 42) :
  ```tsx
  {!user.is_admin && <TelegramConnectionBanner isConnected={telegramStatus.isConnected} />}
  ```
- Le modal, près de la fin du JSX (ancienne ligne 150) :
  ```tsx
  {!user.is_admin && <TelegramConnectionModal isConnected={telegramStatus.isConnected} />}
  ```

**c) `components/assigned-replacements-tab.tsx`** (version actuelle : fonction `formatConfirmationChannel`)

Avant, la confirmation affichait « Telegram » quand `confirmed_via === "telegram"`. Ouvrir la fonction `formatConfirmationChannel` et vérifier qu'elle gère la valeur `"telegram"` en affichant « Telegram ». Si ce cas manque, l'ajouter. Ne pas toucher au reste du fichier.

## 4. Base de données

Le script `024-remove-telegram-from-database.sql` a supprimé les éléments ci-dessous. Ils doivent être recréés avant de restaurer le code, sinon l'application plantera sur des colonnes inexistantes.

### 4.1 Éléments supprimés à recréer

| Élément | Définition d'origine |
| --- | --- |
| `notification_preferences.telegram_chat_id` | `TEXT` |
| `notification_preferences.enable_telegram` | `BOOLEAN DEFAULT false` |
| `users.telegram_required` | `BOOLEAN DEFAULT true` |
| table `telegram_link_codes` | `code VARCHAR(10) PRIMARY KEY`, `user_id INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE`, `chat_id TEXT`, `expires_at TIMESTAMP NOT NULL`, `used_at TIMESTAMP`, `created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP` |

### 4.2 Éléments conservés (à vérifier, mais normalement présents)

- `replacements.confirmed_at` et `replacements.confirmed_via` (script 022).
- Valeur d'enum `telegram_requirement_changed` dans `audit_action_type`.
- Table `manual_notification_deliveries` et colonnes de suivi dans `notifications` (canaux échoués, etc.).

### 4.3 Migration de restauration (à exécuter une seule fois)

```sql
-- Restauration de Telegram
ALTER TABLE notification_preferences
  ADD COLUMN IF NOT EXISTS telegram_chat_id TEXT,
  ADD COLUMN IF NOT EXISTS enable_telegram BOOLEAN DEFAULT false;

ALTER TABLE users
  ADD COLUMN IF NOT EXISTS telegram_required BOOLEAN DEFAULT true;
-- Variante recommandée si on ne veut forcer personne au départ :
-- ALTER TABLE users ALTER COLUMN telegram_required SET DEFAULT false;
-- UPDATE users SET telegram_required = false;

CREATE TABLE IF NOT EXISTS telegram_link_codes (
  code VARCHAR(10) PRIMARY KEY,
  user_id INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  chat_id TEXT,
  expires_at TIMESTAMP NOT NULL,
  used_at TIMESTAMP,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX IF NOT EXISTS idx_telegram_link_codes_user_id ON telegram_link_codes(user_id);
CREATE INDEX IF NOT EXISTS idx_telegram_link_codes_expires_at ON telegram_link_codes(expires_at);
CREATE INDEX IF NOT EXISTS idx_users_telegram_required ON users(telegram_required);

ALTER TYPE audit_action_type ADD VALUE IF NOT EXISTS 'telegram_requirement_changed';
```

À ne PAS réexécuter : `020-sync-telegram-preferences.sql`. Il activait `enable_telegram` pour tous les utilisateurs dont `telegram_required` est vrai ; ce n'est utile que si on veut ce comportement au départ.

### 4.4 Données perdues

Tous les `telegram_chat_id` ont été effacés avec la suppression des colonnes. Après la restauration, chaque pompier doit relier son compte Telegram à nouveau (voir section 7). Décision prise : les anciens identifiants de chat ne seront pas récupérés.

Valeur par défaut de `telegram_required` : `true` oblige tous les non-administrateurs à voir la bannière et le modal de connexion. `false` (recommandé) ne force personne ; l'administrateur active l'exigence par utilisateur depuis la page « Connexions Telegram ».

## 5. Variables d'environnement

| Variable | État | Utilisation |
| --- | --- | --- |
| `TELEGRAM_BOT_TOKEN` | Déjà présente | Envoi des messages, webhook, informations du bot |
| `VERCEL_AUTOMATION_BYPASS_SECRET` | Déjà présente | Ajoutée à l'URL du webhook pour passer la protection de déploiement Vercel |
| `NEXT_PUBLIC_APP_URL` | Non présente | Utilisée pour construire l'URL du webhook et les liens. Le code essaie aussi `VERCEL_URL`. L'ajouter avec l'URL de production (ex. `https://horaire.example.ca`) si le webhook ou les liens pointent au mauvais endroit |
| `VERCEL_ENV` | Automatique | Distingue production et prévisualisation |
| `DATABASE_URL` | Déjà présente | Scripts |

Le nom d'utilisateur du bot n'est pas écrit dans le code : il est lu avec `getBotInfo()` (appel Telegram `getMe`) pour créer le lien `https://t.me/<bot>?start=<CODE>`.

## 6. Plan Mini de restauration fiable

La restauration doit être faite en petites étapes indépendantes. On ne passe à l'étape suivante que si la vérification de l'étape courante est réussie. Chaque étape doit être validée avant toute modification suivante.

### Mini-étape 0 — Geler le point de départ

**Objectif :** pouvoir revenir exactement à l'état actuel.

1. Vérifier la branche et l'état Git.
2. Ne pas commencer si des modifications non liées à Telegram sont présentes sans les identifier.
3. Noter le commit courant.
4. Créer un point de restauration Git avant toute modification.

**Validation obligatoire :** l'état de départ est documenté et le retour arrière est possible.

### Mini-étape 1 — Vérifier le code source historique

**Objectif :** confirmer que la dernière version complète est exploitable.

1. Lire les 13 fichiers Telegram depuis `3833283`.
2. Comparer les fichiers concernés avec les versions actuelles.
3. Vérifier que les imports utilisés par Telegram existent encore dans le projet.
4. Vérifier les changements récents dans `middleware.ts`, `app/dashboard/layout.tsx` et `components/assigned-replacements-tab.tsx`.

**Ne rien modifier.**

**Validation obligatoire :** la liste des fichiers à restaurer et des fichiers à fusionner est confirmée.

### Mini-étape 2 — Sauvegarder la base de données

**Objectif :** protéger les données actuelles avant toute migration.

1. Créer une branche ou une sauvegarde Neon.
2. Vérifier que la sauvegarde contient les tables actuelles.
3. Ne pas tenter de récupérer les anciens `telegram_chat_id`, conformément à la décision prise.

**Validation obligatoire :** la sauvegarde est accessible et son identifiant est noté dans le journal de restauration.

### Mini-étape 3 — Restaurer uniquement le schéma SQL

**Objectif :** remettre les éléments nécessaires avant de charger le code.

1. Exécuter la migration idempotente de la section 4.3.
2. Utiliser `telegram_required = false` au départ pour ne pas forcer tous les utilisateurs.
3. Ne pas exécuter `020-sync-telegram-preferences.sql`.
4. Vérifier les colonnes `telegram_chat_id`, `enable_telegram` et `telegram_required`.
5. Vérifier la table `telegram_link_codes`, ses clés étrangères et ses index.
6. Vérifier la valeur d'audit `telegram_requirement_changed`.

**Validation obligatoire :** toutes les structures existent et aucune donnée existante n'a été supprimée ou modifiée de façon inattendue.

### Mini-étape 4 — Restaurer les fichiers Telegram autonomes

**Objectif :** remettre les fonctions de base sans encore reconnecter toute l'interface.

Restaurer les fichiers de la section 3.1, par petits groupes :

1. `lib/telegram.ts`;
2. les actions Telegram;
3. les routes `/api/telegram/*`;
4. les composants Telegram;
5. les pages de statut et de webhook;
6. le script de configuration du webhook.

Après chaque groupe :

- vérifier les imports;
- rechercher les chemins inexistants;
- lancer le type-check ciblé ou le build si nécessaire;
- corriger uniquement les erreurs de ce groupe.

**Validation obligatoire :** les actions et routes Telegram sont compilables avant de restaurer les notifications.

### Mini-étape 5 — Restaurer les notifications

**Objectif :** réintégrer Telegram sans casser les autres canaux.

Restaurer les fichiers de la section 3.2 dans cet ordre :

1. `app/actions/notifications.tsx`;
2. `app/actions/send-manual-notification.ts`;
3. `app/actions/send-assignment-notification.tsx`;
4. `components/notification-preferences-form.tsx`;
5. `components/send-notification-form.tsx`;
6. `app/dashboard/page.tsx`;
7. `app/dashboard/settings/page.tsx`;
8. `app/dashboard/settings/notification-history/[id]/page.tsx`.

Après chaque groupe, vérifier que les notifications internes et les courriels restent fonctionnels. Ne pas restaurer les anciennes versions si le diff contient un changement récent qui n'est pas lié à Telegram.

**Validation obligatoire :** aucune erreur TypeScript et aucun canal non-Telegram n'est perdu.

### Mini-étape 6 — Fusionner les fichiers sensibles

**Objectif :** réintégrer Telegram sans écraser les corrections récentes.

1. Dans `middleware.ts`, ajouter seulement l'exception du webhook et l'exclusion correspondante du matcher.
2. Dans `app/dashboard/layout.tsx`, ajouter seulement le statut Telegram, la bannière et le modal.
3. Dans `components/assigned-replacements-tab.tsx`, conserver `formatConfirmationChannel` et ajouter uniquement le cas `telegram` s'il manque.
4. Relire le diff de ces trois fichiers.

**Validation obligatoire :** les redirections, la session Edge, le mot de passe forcé et les changements de remplacements sont toujours présents.

### Mini-étape 7 — Vérification statique complète

**Objectif :** trouver les erreurs avant tout déploiement.

1. Lancer le type-check.
2. Lancer le lint si le projet le permet.
3. Lancer le build de production.
4. Vérifier les imports serveur/client et les routes API.
5. Vérifier qu'aucun ancien secret ou jeton Telegram n'est écrit en dur.
6. Vérifier que `TELEGRAM_BOT_TOKEN` est lu uniquement côté serveur.

**Validation obligatoire :** type-check, lint et build réussissent, ou chaque exception est documentée avant de continuer.

### Mini-étape 8 — Vérification de l'interface

**Objectif :** vérifier les parcours visibles avant de configurer le webhook.

Tester dans la prévisualisation ou l'environnement de test :

1. page des préférences de notification;
2. génération du lien Telegram;
3. page des connexions Telegram administrateur;
4. page de configuration du webhook;
5. historique des notifications;
6. affichage des remplacements confirmés par Telegram.

**Validation obligatoire :** aucune page ne produit d'erreur console ou serveur liée à Telegram.

### Mini-étape 9 — Déploiement et webhook

**Objectif :** connecter Telegram uniquement après validation du code.

1. Déployer la version validée.
2. Vérifier que l'URL publique est celle de production.
3. Configurer le webhook depuis `/dashboard/settings/telegram-webhook`.
4. Vérifier la réponse Telegram et l'URL enregistrée.
5. Vérifier que Telegram ne reçoit pas de redirection 307 vers `/login`.

**Validation obligatoire :** le webhook est actif et répond avec un statut HTTP réussi.

### Mini-étape 10 — Tests fonctionnels par scénarios

**Objectif :** valider toutes les fonctions, pas seulement la compilation.

Faire les tests de la section 7 dans l'ordre. Utiliser un compte de test et relier manuellement son compte Telegram, puisque les anciens chat IDs ne sont pas restaurés.

**Validation obligatoire :** chaque scénario est noté comme réussi ou bloqué avec la cause exacte.

### Mini-étape 11 — Nettoyage et clôture

**Objectif :** laisser une restauration propre et reproductible.

1. Retirer les logs de diagnostic temporaires.
2. Vérifier que les migrations ne sont pas dupliquées.
3. Vérifier le diff final.
4. Mettre à jour ce document si une étape réelle diffère.
5. Créer un commit dédié à la restauration Telegram.
6. Conserver l'identifiant de la sauvegarde et les résultats des tests.

**Règle d'arrêt :** si une étape échoue, on s'arrête, on conserve les résultats, on corrige uniquement la cause identifiée et on ne passe pas à l'étape suivante sans nouvelle validation.

## 7. Tests de validation

1. **Liaison** : se connecter avec un compte de test non administrateur, ouvrir les préférences de notification, cliquer sur « Connecter Telegram », ouvrir le lien `t.me`, envoyer `/start CODE`. Le bot doit confirmer, et `notification_preferences.telegram_chat_id` doit contenir l'identifiant du chat.
2. **Envoi d'une notification** : créer une assignation ou une notification manuelle vers ce compte. Le message doit arriver dans Telegram, et l'historique doit afficher « Telegram » comme canal réussi.
3. **Bouton de confirmation** : appuyer sur le bouton dans le message. Le popup de Telegram doit s'afficher, et `replacements.confirmed_at` et `confirmed_via = 'telegram'` doivent être renseignés. L'onglet des remplacements assignés doit afficher « Telegram ».
4. **Déconnexion** : le bouton de déconnexion doit vider `telegram_chat_id`.
5. **Page « Connexions Telegram »** : la liste doit afficher le bon état par utilisateur, et le bouton d'exigence doit mettre à jour `telegram_required` et écrire une entrée d'audit `telegram_requirement_changed`.
6. **Utilisateur non connecté à Telegram** : la bannière et le modal doivent apparaître seulement pour les non-administrateurs avec `telegram_required = true`.
7. **Canal désactivé** : décocher Telegram dans les préférences ne doit plus envoyer de message.
8. **Non-régression** : mot de passe forcé, redirections de connexion, courriels et notifications dans l'application doivent fonctionner comme avant.

## 8. Points de vigilance

- Le webhook sans exception dans le middleware donne des erreurs 307 vers `/login` côté Telegram : les boutons de confirmation ne fonctionnent pas. Vérifier l'ajout de la section 3.3 a).
- Le fuseau horaire : plusieurs correctifs en 2026 concernent l'heure affichée dans le message de confirmation (`fcbe8df`, `fb011b8` et autres). Le code restauré les contient déjà ; ne pas les refaire.
- Si une colonne manque (erreur `column ... does not exist`), la migration de la section 4.3 n'a pas été exécutée sur la bonne base.
- Si la protection de déploiement Vercel bloque le webhook, vérifier `VERCEL_AUTOMATION_BYPASS_SECRET` et réenregistrer le webhook.
- Si un fichier restauré a changé depuis cette rédaction, refaire la comparaison de la section 3.2 avant de l'écraser.

## 9. Annulation (retour à l'état sans Telegram)

Réexécuter `scripts/024-remove-telegram-from-database.sql`, puis supprimer les fichiers restaurés ou rétablir les versions précédentes de Git (`git checkout <commit-avant-restauration> -- <fichiers>`).
