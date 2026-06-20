# Guide de Sécurité Backend & Architecture (Pour Débutants)

Ce document a été créé pour vous aider à vous souvenir de la procédure exacte que nous avons mise en place pour sécuriser votre site ELY EATS, et surtout pour comprendre **pourquoi** nous l'avons fait.

---

## 1. Comprendre le problème : Frontend vs Backend

Pour bien comprendre la sécurité, il faut séparer votre site en deux mondes :

- **Le Frontend (Le navigateur du client)** : C'est tout ce qui s'affiche sur l'écran de l'utilisateur (React, le code HTML, les boutons, le chat). **Règle d'or : Tout ce qui est dans le Frontend est public.** Si vous mettez un mot de passe ou une URL secrète ici, un utilisateur peut faire un clic droit > "Inspecter" et le lire.
- **Le Backend (Le Serveur)** : C'est l'ordinateur distant (comme les serveurs de Vercel) qui fait les calculs lourds. **Règle d'or : Le Backend est privé.** Personne ne peut voir le code qui tourne ici, sauf vous.

### Notre problème initial
Au début, votre formulaire de commande et votre chat envoyaient la requête **directement** depuis le Frontend vers vos workflows n8n.
*Le danger :* N'importe qui pouvait voir l'URL de votre n8n dans son navigateur et envoyer de fausses commandes ou du spam avec un simple script.

---

## 2. Qu'est-ce que TanStack Start ?

Pour résoudre ce problème, nous avons utilisé **TanStack Start**.
Normalement, React est uniquement un outil Frontend. Mais **TanStack Start** est ce qu'on appelle un framework *Full-Stack*. Cela veut dire qu'il permet à votre code React d'avoir son propre **Backend intégré**. 

Grâce à TanStack Start, nous pouvons écrire des **Fonctions Serveur** (`createServerFn`). Ces fonctions ressemblent à du code normal, mais elles garantissent qu'elles ne s'exécuteront **jamais** sur le navigateur du client, uniquement sur les serveurs sécurisés de Vercel.

---

## 3. La Solution : Le "Backend Proxy"

Nous avons transformé votre propre serveur en un **Proxy** (un intermédiaire).

> **Avant :** Navigateur du Client ➔ ➔ ➔ n8n (Pas sécurisé)
> **Maintenant :** Navigateur du Client ➔ Serveur Vercel ➔ n8n (Sécurisé)

Le client demande poliment à votre serveur Vercel de passer la commande. Votre serveur (qui connaît l'URL secrète) transmet la commande à n8n, puis renvoie la réponse au client. Le client ne voit jamais l'URL de n8n !

---

## 4. La Procédure Étape par Étape que nous avons effectuée

Si jamais vous devez refaire cela pour un autre projet, voici la recette exacte :

### Étape A : Cacher les variables (Le `.env`)
Dans l'écosystème web (Vite), si une variable commence par `VITE_`, elle est automatiquement envoyée au Frontend (public). 
1. Nous avons ouvert le fichier `.env`.
2. Nous avons renommé `VITE_ORDER_WEBHOOK_URL` en `N8N_ORDER_WEBHOOK_URL` (en enlevant le "VITE_").
3. *Résultat :* La variable est maintenant strictement bloquée dans le Backend.

### Étape B : Créer les Fonctions Serveur
Nous avons créé un fichier `src/lib/server-functions.ts`.
Dedans, nous avons utilisé `createServerFn` de TanStack Start :
```typescript
import { createServerFn } from "@tanstack/react-start";

export const submitOrderFn = createServerFn({ method: "POST" })
  .handler(async ({ data }) => {
    // 1. On lit l'URL secrète depuis le serveur
    const url = process.env.N8N_ORDER_WEBHOOK_URL;
    
    // 2. Le SERVEUR envoie les données à n8n
    const res = await fetch(url, { method: "POST", body: JSON.stringify(data) });
    return await res.json();
  });
```

### Étape C : Modifier les composants React (Frontend)
Dans `order-modal.tsx` et `chat-widget.tsx`, nous avons retiré les appels directs via `fetch(URL)`.
Nous les avons remplacés par un appel à notre fonction serveur :
```typescript
import { submitOrderFn } from "@/lib/server-functions";

// Le navigateur demande simplement au serveur de s'en occuper
await submitOrderFn({ data: votreCommande });
```

### Étape D : Déploiement et Configuration Vercel/Lovable
Puisque le code ne contient plus les URLs, Vercel et Lovable ont besoin qu'on leur donne ces secrets.
1. Nous sommes allés dans Vercel > Settings > Environment Variables.
2. Nous avons ajouté `N8N_ORDER_WEBHOOK_URL` et la valeur.
3. Nous avons redéployé le site.

> [!TIP]
> **Pourquoi le plugin Nitro ?**
> Pour que tout cela fonctionne sur Vercel, nous avons dû ajouter `nitro: true` dans le fichier `vite.config.ts`. Nitro est le moteur interne de TanStack Start qui transforme ce code en un vrai serveur compatible avec Vercel !
