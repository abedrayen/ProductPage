# Configuration Google Sheets pour les Commandes

Ce guide vous explique comment configurer Google Sheets pour sauvegarder automatiquement les commandes.

## Option 1 : Google Apps Script (Recommandé)

### Étape 1 : Créer un Google Sheet

1. Allez sur [Google Sheets](https://sheets.google.com)
2. Créez un nouveau tableur
3. Nommez la première feuille "Commandes"
4. Ajoutez les en-têtes suivants dans la première ligne (A1 à I1) :
   ```
   Date/Heure | Nom | Email | Téléphone | Ville | Adresse | Notes | Produit | Prix
   ```

### Étape 2 : Créer le Script Google Apps Script

1. Dans votre Google Sheet, allez dans **Extensions** > **Apps Script**
2. Supprimez tout le code existant et collez ce code :

```javascript
function doPost(e) {
  try {
    const sheet = SpreadsheetApp.getActiveSpreadsheet().getSheetByName('Commandes');
    
    // Parse form data from POST request
    let data = {};
    
    // Method 1: Try to get from e.parameter (works for GET and form submissions)
    if (e.parameter && Object.keys(e.parameter).length > 0) {
      data = {
        timestamp: e.parameter.timestamp || new Date().toISOString(),
        name: e.parameter.name || '',
        email: e.parameter.email || '',
        phone: e.parameter.phone || '',
        city: e.parameter.city || '',
        address: e.parameter.address || '',
        notes: e.parameter.notes || '',
        product: e.parameter.product || '',
        price: e.parameter.price || ''
      };
    }
    // Method 2: Parse from postData.contents (for application/x-www-form-urlencoded)
    else if (e.postData && e.postData.contents) {
      const contents = e.postData.contents;
      
      // Try to parse as JSON first
      try {
        data = JSON.parse(contents);
      } catch (e) {
        // If not JSON, parse as URL-encoded form data
        const params = contents.split('&');
        params.forEach(param => {
          const [key, value] = param.split('=');
          if (key && value) {
            data[decodeURIComponent(key)] = decodeURIComponent(value.replace(/\+/g, ' '));
          }
        });
      }
    }
    
    // Ensure we have at least some data
    if (!data.name && !data.email && !data.phone) {
      // Log for debugging
      Logger.log('No data received. e.parameter: ' + JSON.stringify(e.parameter));
      Logger.log('e.postData: ' + JSON.stringify(e.postData));
      return ContentService.createTextOutput('No data received');
    }
    
    // Prepare row data
    const row = [
      data.timestamp || new Date().toLocaleString('fr-FR'),
      data.name || '',
      data.email || '',
      data.phone || '',
      data.city || '',
      data.address || '',
      data.notes || '',
      data.product || '',
      data.price || ''
    ];
    
    // Append to sheet
    sheet.appendRow(row);
    
    // Log success for debugging
    Logger.log('Order saved: ' + JSON.stringify(data));
    
    // Return simple text response
    return ContentService.createTextOutput('OK');
    
  } catch (error) {
    // Log error for debugging
    Logger.log('Error: ' + error.toString());
    Logger.log('Stack: ' + error.stack);
    return ContentService.createTextOutput('Error: ' + error.toString());
  }
}

// Handle GET requests (for testing)
function doGet(e) {
  return HtmlService.createHtmlOutput('<p>Service de commandes actif. Utilisez POST pour envoyer des commandes.</p>');
}
```

3. Cliquez sur **Enregistrer** (icône disquette)
4. Donnez un nom au projet (ex: "OrderHandler")

### Étape 3 : Déployer comme Web App

1. Cliquez sur **Déployer** > **Nouveau déploiement**
2. Cliquez sur l'icône d'engrenage ⚙️ à côté de "Type" et sélectionnez **Application Web**
3. Configurez :
   - **Description** : "API pour recevoir les commandes"
   - **Exécuter en tant que** : Moi (votre email)
   - **Qui a accès** : Tous
4. Cliquez sur **Déployer**
5. **IMPORTANT** : Autorisez les permissions quand Google vous le demande
6. Copiez l'**URL du déploiement** (elle ressemble à : `https://script.google.com/macros/s/...`)

### Étape 4 : Configurer dans index.html

1. Ouvrez `index.html`
2. Trouvez la ligne avec `const GOOGLE_SHEETS_WEB_APP_URL = 'YOUR_GOOGLE_APPS_SCRIPT_URL_HERE';`
3. Remplacez `YOUR_GOOGLE_APPS_SCRIPT_URL_HERE` par l'URL que vous avez copiée
4. Sauvegardez le fichier

## Option 2 : Utilisation de localStorage (Temporaire)

Si vous ne configurez pas Google Sheets immédiatement, les commandes seront automatiquement sauvegardées dans le localStorage du navigateur. Vous pouvez les récupérer en ouvrant la console du navigateur (F12) et en tapant :

```javascript
JSON.parse(localStorage.getItem('orders'))
```

## Test

1. Ouvrez votre page `index.html` dans un navigateur
2. Cliquez sur n'importe quel bouton "Je le veux" ou "Commander"
3. Remplissez le formulaire
4. Soumettez la commande
5. Vérifiez dans votre Google Sheet que la commande a été enregistrée

## Dépannage et Débogage

### Vérifier que les données sont reçues

1. **Vérifier les logs Google Apps Script** :
   - Allez dans votre Google Sheet
   - **Extensions** > **Apps Script**
   - Cliquez sur **Exécutions** (icône d'horloge) dans le menu de gauche
   - Vous verrez toutes les exécutions récentes avec les logs

2. **Vérifier les données dans la console du navigateur** :
   - Ouvrez la console (F12)
   - Vous devriez voir : `Form submitted to Google Sheets with data: {...}`
   - Vérifiez aussi localStorage : `JSON.parse(localStorage.getItem('orders'))`

3. **Tester le script directement** :
   - Ouvrez l'URL de votre Web App dans le navigateur
   - Vous devriez voir : "Service de commandes actif. Utilisez POST pour envoyer des commandes."

### Problèmes courants

- **Les commandes ne s'enregistrent pas** :
  - Vérifiez que l'URL dans `index.html` est correcte
  - Vérifiez que le script est déployé avec "Tous" comme accès
  - Vérifiez les logs dans Google Apps Script (Exécutions)
  - Assurez-vous que la feuille "Commandes" existe dans votre Google Sheet

- **Erreur de permissions** :
  - Assurez-vous d'avoir autorisé toutes les permissions demandées par Google Apps Script
  - Allez dans **Déployer** > **Gérer les déploiements** > **Modifier** et réautorisez les permissions

- **Les données sont dans localStorage mais pas dans Google Sheets** :
  - C'est normal, localStorage est un système de secours
  - Vérifiez les logs Google Apps Script pour voir si les données sont reçues
  - Vérifiez que le nom de la feuille est exactement "Commandes" (sensible à la casse)

- **Erreur "No data received" dans les logs** :
  - Le script reçoit la requête mais ne trouve pas les données
  - Vérifiez que le formulaire envoie bien les données en POST
  - Vérifiez la console du navigateur pour voir les données envoyées

## Alternative : Utiliser un service tiers

Si vous préférez utiliser un autre service, vous pouvez modifier la fonction `submitOrder()` dans `index.html` pour envoyer les données à :
- Airtable
- Zapier
- Make.com
- Votre propre API backend

