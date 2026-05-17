# Liste Complète des Tâches - Milestone 3 IDS

## 📋 PHASE 1 : PRÉPARATION & CONFIGURATION

### Tâche 1.1 : Configuration du Logging NGINX

* [ ] Configurer le reverse proxy NGINX pour logs détaillés
* [ ] Modifier `/etc/nginx/nginx.conf` pour format de log personnalisé
* [ ] Inclure : IP source, timestamp, URL, méthode HTTP, code réponse, user-agent, taille
* [ ] Tester que les logs s'écrivent correctement dans `/var/log/nginx/access.log`
* [ ] Vérifier la rotation des logs (optionnel)

**Exemple de configuration :**

```nginx
log_format detailed '$remote_addr - $remote_user [$time_local] '
                    '"$request" $status $body_bytes_sent '
                    '"$http_referer" "$http_user_agent"';
access_log /var/log/nginx/access.log detailed;

```

---

## 📋 PHASE 2 : DÉVELOPPEMENT DES SCRIPTS

### Tâche 2.1 : Script d'Activité Normale

* [ ] Créer `normal_activity.sh` ou `.py`
* [ ] Simuler navigation normale (GET pages principales)
* [ ] Simuler téléchargements légitimes
* [ ] Espacer les requêtes (délais réalistes)
* [ ] Tester et valider

**Exemple de requêtes à simuler :**

```bash
# Requêtes normales espacées
curl http://reverse-proxy/
curl http://reverse-proxy/about
curl http://reverse-proxy/contact

```

### Tâche 2.2 : Script(s) d'Activité Malveillante

Créer plusieurs scripts d'attaque :

#### Script 2.2a : Scan de Répertoires

* [ ] Créer `attack_directory_scan.sh`
* [ ] Tester URLs communes : `/admin`, `/backup`, `/.git`, `/config`, etc.
* [ ] Générer beaucoup de 404
* [ ] Tester et valider

#### Script 2.2b : Brute Force

* [ ] Créer `attack_bruteforce.sh`
* [ ] Tentatives répétées sur `/login` ou `/admin`
* [ ] Différents mots de passe
* [ ] Générer des 401/403
* [ ] Tester et valider

#### Script 2.2c : Tentatives d'Exploitation

* [ ] Créer `attack_exploit.sh`
* [ ] Injecter patterns suspects dans URLs : `?id=1' OR '1'='1`
* [ ] Tester XSS : `?search=<script>alert(1)</script>`
* [ ] Path traversal : `?file=../../etc/passwd`
* [ ] Tester et valider

#### Script 2.2d : Flood/DoS Simple

* [ ] Créer `attack_flood.sh`
* [ ] Envoyer nombreuses requêtes rapides depuis une IP
* [ ] Tester et valider

### Tâche 2.3 : Script d'Analyse Post-Mortem

* [ ] Créer `analyze_logs.sh` ou `.py`
* [ ] **Fonctionnalité 1 :** Compter requêtes par IP
* [ ] **Fonctionnalité 2 :** Détecter scans (nombreux 404)
* [ ] **Fonctionnalité 3 :** Détecter brute force (nombreux 401/403)
* [ ] **Fonctionnalité 4 :** Détecter patterns d'exploitation (union, select, script, ..)
* [ ] **Fonctionnalité 5 :** Détecter flood (trop de requêtes/sec)
* [ ] Générer rapport structuré
* [ ] Tester sur logs générés

**Structure du rapport :**

```text
=== RAPPORT D'ANALYSE IDS ===
Période analysée: ...
Total requêtes: ...

[1] IPs Suspectes (top 10)
[2] Scans Détectés (404 multiples)
[3] Tentatives Brute Force
[4] Patterns d'Exploitation
[5] Activités de Flood

```

### Tâche 2.4 : Script de Monitoring en Temps Réel

* [ ] Créer `monitor_realtime.sh` ou `.py`
* [ ] Utiliser `tail -f` ou équivalent Python
* [ ] Détecter patterns en temps réel
* [ ] Afficher alertes avec couleurs (🚨 🔴 ⚠️)
* [ ] Catégoriser les alertes par gravité
* [ ] Tester en parallèle avec scripts d'attaque

**Fonctionnalités :**

```bash
🟢 Requête normale
⚠️  ALERTE: Scan détecté
🚨 CRITIQUE: Tentative d'exploitation
🔴 FLOOD: 50 requêtes en 10 sec

```

### Tâche 2.5 : Script de Génération de Rapport Final (optionnel)

* [ ] Créer `generate_report.sh`
* [ ] Compiler statistiques complètes
* [ ] Générer graphiques ASCII ou HTML (optionnel)
* [ ] Exporter en format lisible

---

## 📋 PHASE 3 : TESTS ET VALIDATION

### Tâche 3.1 : Tests Individuels

* [ ] Tester chaque script d'attaque individuellement
* [ ] Vérifier que les logs sont générés correctement
* [ ] Tester l'analyse post-mortem sur chaque type d'attaque
* [ ] Ajuster les seuils de détection si nécessaire

### Tâche 3.2 : Test Intégré

* [ ] Lancer monitoring en temps réel
* [ ] Exécuter activité normale → vérifier pas de fausses alertes
* [ ] Exécuter chaque attaque → vérifier détection
* [ ] Générer rapport final → vérifier cohérence

### Tâche 3.3 : Préparation Wireshark (optionnel mais recommandé)

* [ ] Capturer trafic avec Wireshark pendant les scénarios
* [ ] Filtrer et préparer captures pertinentes
* [ ] Corréler avec les logs

---

## 📋 PHASE 4 : DOCUMENTATION ET ANALYSE

### Tâche 4.1 : Documentation Technique

* [ ] Documenter configuration NGINX
* [ ] Documenter chaque script (usage, paramètres)
* [ ] Créer README avec instructions d'exécution
* [ ] Documenter les indicateurs de détection choisis

### Tâche 4.2 : Analyse Théorique

* [ ] Expliquer comment les logs supportent un IDS
* [ ] Identifier les types d'attaques détectables
* [ ] Lister les limitations de l'approche
* [ ] Proposer améliorations possibles

### Tâche 4.3 : Justification des Choix

* [ ] Pourquoi ces indicateurs spécifiques ?
* [ ] Pourquoi ces seuils de détection ?
* [ ] Quels compromis (faux positifs vs faux négatifs) ?
* [ ] Pourquoi cette architecture ?

---

## 📋 PHASE 5 : PRÉPARATION DE LA PRÉSENTATION

### Tâche 5.1 : Création des Slides

* [ ] Slide 1 : Choix de l'IDS - Motivation
* [ ] Slide 2 : Architecture et configuration
* [ ] Slide 3 : Indicateurs de détection choisis
* [ ] Slide 4 : Scénarios de test
* [ ] Slide 5 : Résultats et analysis
* [ ] Slide 6 : Améliorations de sécurité et limitations
* [ ] Slide 7 : Conclusion

### Tâche 5.2 : Préparation de la Démo

* [ ] Préparer 3-4 terminaux/fenêtres :
* **Terminal 1 :** Monitoring temps réel
* **Terminal 2 :** Exécution des scripts
* **Terminal 3 :** Logs bruts (optionnel)
* **Navigateur :** Wireshark (optionnel)


* [ ] Tester le flow complet de la démo
* [ ] Chronométrer (doit tenir en ~5 min max)
* [ ] Préparer plan B si monitoring temps réel échoue

### Tâche 5.3 : Scénario de Démonstration

Créer un script de présentation minute par minute :

* **[0:00-0:30]** Introduction + motivation du choix
* **[0:30-1:30]** Explication architecture et configuration
* **[1:30-2:00]** Lancement monitoring temps réel
* **[2:00-3:00]** Scénario 1: Activité normale (pas d'alerte)
* **[3:00-5:00]** Scénario 2: Attaque (alertes en direct!)
* **[5:00-7:00]** Analyse post-mortem et rapport
* **[7:00-8:30]** Explication lien avec IDS + améliorations
* **[8:30-10:00]** Limitations et questions

### Tâche 5.4 : Répétition

* [ ] Faire au moins 2 répétitions complètes
* [ ] Chronométrer chaque partie
* [ ] Identifier points à améliorer
* [ ] Préparer réponses aux questions probables

---

## 📋 PHASE 6 : LIVRABLES FINAUX

### Tâche 6.1 : Code et Configuration

* [ ] Tous les scripts commentés et fonctionnels
* [ ] Configuration NGINX documentée
* [ ] README.md avec instructions complètes
* [ ] Logs d'exemple (avant/après attaque)

### Tâche 6.2 : Documentation

* [ ] Description du choix IDS et motivation
* [ ] Architecture implémentée
* [ ] Justification des choix de design
* [ ] Rapport d'analyse des logs
* [ ] Améliorations et limitations

### Tâche 6.3 : Présentation

* [ ] Slides finalisés (PDF)
* [ ] Démo préparée et testée
* [ ] Backup plan en cas de problème technique
* [ ] Questions/réponses anticipées

---

## 🎯 CHECKLIST FINALE AVANT LA PRÉSENTATION

* [ ] Topology GNS3 fonctionnelle
* [ ] Reverse proxy accessible et logs activés
* [ ] Tous les scripts testés et fonctionnels
* [ ] Monitoring temps réel opérationnel
* [ ] Slides prêts
* [ ] Démo chronométrée (<10 min)
* [ ] Captures Wireshark prêtes (optionnel)
* [ ] Backup des logs et résultats
* [ ] Plan B si problème technique

---

## ⏱️ ESTIMATION DU TEMPS

| Phase | Durée Estimée |
| --- | --- |
| **Phase 1 : Configuration** | 2-3 heures |
| **Phase 2 : Scripts** | 6-8 heures |
| **Phase 3 : Tests** | 3-4 heures |
| **Phase 4 : Documentation** | 3-4 heures |
| **Phase 5 : Présentation** | 4-5 heures |
| **TOTAL** | **18-24 heures** |
