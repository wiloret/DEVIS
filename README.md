<!DOCTYPE html>
<html lang="fr">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Calculateur de Devis Interactif</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <script src="https://cdn.sheetjs.com/xlsx-0.20.2/package/dist/xlsx.full.min.js"></script>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/jspdf/2.5.1/jspdf.umd.min.js"></script>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/jspdf-autotable/3.8.2/jspdf.plugin.autotable.min.js"></script>
    <link rel="preconnect" href="https://fonts.googleapis.com">
    <link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
    <link href="https://fonts.googleapis.com/css2?family=Inter:wght@400;500;600;700;800;900&display=swap" rel="stylesheet">
    <style>
        body { 
            font-family: 'Inter', sans-serif; 
            background-color: #f8fafc; /* gray-50 */
        }
        .wizard-step { display: none; }
        .wizard-step.active { display: block; }
        .progress-bar-fill {
            transition: width 0.5s ease-in-out;
        }
        .product-card {
            transition: transform 0.2s ease-in-out, box-shadow 0.2s ease-in-out;
        }
        .product-card:hover {
            transform: translateY(-5px);
            box-shadow: 0 10px 15px -3px rgb(0 0 0 / 0.1), 0 4px 6px -4px rgb(0 0 0 / 0.1);
        }
        .modal {
            display: none;
            transition: opacity 0.3s ease;
        }
        .modal.is-open {
            display: flex;
            opacity: 1;
        }
        .modal-content {
            transition: transform 0.3s ease;
            transform: scale(0.95);
        }
        .modal.is-open .modal-content {
            transform: scale(1);
        }
        .toast {
            transition: opacity 0.5s, transform 0.5s;
            transform: translateX(100%);
            opacity: 0;
        }
        .toast.show {
            transform: translateX(0);
            opacity: 1;
        }
        /* Custom scrollbar for summary */
        .summary-scroll::-webkit-scrollbar {
            width: 6px;
        }
        .summary-scroll::-webkit-scrollbar-track {
            background: #f1f5f9; /* gray-100 */
        }
        .summary-scroll::-webkit-scrollbar-thumb {
            background: #94a3b8; /* gray-400 */
            border-radius: 10px;
        }
        .summary-scroll::-webkit-scrollbar-thumb:hover {
            background: #64748b; /* gray-500 */
        }
        /* Animations */
        @keyframes fadeIn {
            from { opacity: 0; transform: translateY(10px); }
            to { opacity: 1; transform: translateY(0); }
        }
        .fade-in {
            animation: fadeIn 0.5s ease-in-out forwards;
        }
    </style>
</head>
<body class="bg-slate-50">

    <!-- Inputs cachés pour les imports/exports -->
    <input type="file" id="load-order-input" class="hidden" accept=".json">
    <input type="file" id="import-licensees-input" class="hidden" accept=".xlsx, .xls">
    <input type="file" id="import-stock-input" class="hidden" accept=".json">
    <input type="file" id="import-club-range-input" class="hidden" accept=".json">

    <!-- Conteneur pour les notifications (toast) -->
    <div id="toast-container" class="fixed top-5 right-5 z-[100] space-y-3 w-80"></div>

    <!-- Modale principale réutilisable -->
    <div id="main-modal" class="modal fixed inset-0 bg-black bg-opacity-60 z-50 justify-center items-center p-4 opacity-0">
        <div class="modal-content bg-white rounded-lg shadow-xl p-6 w-full max-w-md relative">
            <button id="main-modal-close-btn" class="absolute top-3 right-3 text-gray-400 hover:text-gray-600">
                <svg class="h-6 w-6" fill="none" viewBox="0 0 24 24" stroke="currentColor"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M6 18L18 6M6 6l12 12"></path></svg>
            </button>
            <h3 id="main-modal-title" class="text-xl font-bold text-gray-800 mb-4">Titre du Modal</h3>
            <div id="main-modal-body" class="text-gray-600 mb-6 max-h-[60vh] overflow-y-auto"></div>
            <div id="main-modal-actions" class="flex justify-end space-x-3"></div>
        </div>
    </div>

    <!-- Conteneur principal de l'application -->
    <div class="container mx-auto p-4 sm:p-6 lg:p-8">
        
        <header class="mb-6 text-center">
            <h1 class="text-4xl font-extrabold text-gray-800 tracking-tight">Calculateur de Devis</h1>
            <p class="mt-2 text-lg text-gray-500">Un outil simple pour créer vos devis.</p>
            <p id="autosave-status" class="mt-1 text-xs text-gray-400" style="min-height: 1em;"></p>
        </header>

        <!-- Barre de progression du Wizard -->
        <div id="wizard-progress" class="w-full bg-gray-200 rounded-full h-2.5 mb-8">
            <div id="wizard-progress-bar" class="progress-bar-fill bg-indigo-600 h-2.5 rounded-full" style="width: 33%"></div>
        </div>

        <!-- Étape 1: Informations Générales -->
        <div id="step-1" class="wizard-step active fade-in">
             <div class="bg-white p-8 rounded-xl shadow-lg max-w-3xl mx-auto">
                <h2 class="text-2xl font-bold text-gray-800 border-b pb-3 mb-6">Étape 1: Informations Générales</h2>
                <div class="grid grid-cols-1 md:grid-cols-2 gap-6">
                    <div>
                        <label for="clubName" class="block text-sm font-medium text-gray-700">Nom du Club / Client <span class="text-red-500">*</span></label>
                        <input type="text" id="clubName" list="club-list" class="mt-1 block w-full rounded-md border-gray-300 shadow-sm focus:border-indigo-500 focus:ring-indigo-500">
                        <datalist id="club-list"></datalist>
                    </div>
                    <div>
                        <label for="departement" class="block text-sm font-medium text-gray-700">Département <span class="text-red-500">*</span></label>
                        <input type="text" id="departement" class="mt-1 block w-full rounded-md border-gray-300 shadow-sm focus:border-indigo-500 focus:ring-indigo-500">
                    </div>
                    <div>
                        <label for="clientNumber" class="block text-sm font-medium text-gray-700">N° Client</label>
                        <input type="text" id="clientNumber" list="client-list" class="mt-1 block w-full rounded-md border-gray-300 shadow-sm focus:border-indigo-500 focus:ring-indigo-500">
                        <datalist id="client-list"></datalist>
                    </div>
                    <div>
                        <label for="orderDate" class="block text-sm font-medium text-gray-700">Date du devis</label>
                        <input type="date" id="orderDate" class="mt-1 block w-full rounded-md border-gray-300 shadow-sm focus:border-indigo-500 focus:ring-indigo-500">
                    </div>
                </div>
                <div class="mt-8 pt-6 border-t flex justify-end">
                    <button id="next-to-step-2" class="px-8 py-3 bg-indigo-600 text-white font-bold rounded-md hover:bg-indigo-700 shadow-md focus:outline-none focus:ring-2 focus:ring-offset-2 focus:ring-indigo-500">Continuer</button>
                </div>
            </div>
        </div>

        <!-- Étape 2: Sélection des Articles -->
        <div id="step-2" class="wizard-step fade-in">
            <div class="grid grid-cols-1 lg:grid-cols-3 gap-8">
                <!-- Colonne de gauche : Sélection des produits -->
                <div class="lg:col-span-2 bg-white p-6 rounded-xl shadow-lg">
                    <h2 class="text-2xl font-bold text-gray-800 mb-4">Étape 2: Sélection des Articles</h2>
                     <!-- Tabs pour les catégories de produits -->
                    <div id="product-tabs-container" class="flex border-b border-gray-200 mb-4">
                        <button data-tab="CYCLISME/RUNNING" class="product-tab-btn py-2 px-4 -mb-px font-medium text-sm border-b-2 border-indigo-500 text-indigo-600">CYCLISME/RUNNING</button>
                        <button data-tab="Accessoires" class="product-tab-btn py-2 px-4 -mb-px font-medium text-sm text-gray-500 hover:text-gray-700">Accessoires</button>
                        <button data-tab="GAMME ENFANTS" class="product-tab-btn py-2 px-4 -mb-px font-medium text-sm text-gray-500 hover:text-gray-700">GAMME ENFANTS</button>
                    </div>
                    <!-- Grille des produits -->
                    <div id="product-grid" class="grid grid-cols-1 md:grid-cols-2 xl:grid-cols-3 gap-4 h-[60vh] overflow-y-auto pr-2 summary-scroll">
                        <!-- Les cartes produits seront injectées ici par JS -->
                    </div>
                </div>
                <!-- Colonne de droite : Récapitulatif du devis -->
                <div class="lg:col-span-1">
                    <div class="bg-white p-6 rounded-xl shadow-lg sticky top-8">
                        <h3 class="text-xl font-bold text-gray-800 border-b pb-3 mb-4">Récapitulatif du Devis</h3>
                        <div id="order-summary-container" class="space-y-3 h-[60vh] overflow-y-auto summary-scroll pr-2">
                            <p id="empty-cart-message" class="text-gray-500 text-center py-10">Votre devis est vide.</p>
                            <!-- Les articles seront injectés ici -->
                        </div>
                        <div class="mt-4 pt-4 border-t">
                             <div class="flex justify-between items-center text-lg font-bold">
                                <span>Total TTC</span>
                                <span id="summary-grand-total-ttc">0.00€</span>
                            </div>
                        </div>
                    </div>
                </div>
            </div>
             <div class="mt-8 pt-6 flex justify-between items-center">
                <button id="back-to-step-1" class="px-6 py-2 bg-gray-200 text-gray-700 font-medium rounded-md hover:bg-gray-300">Précédent</button>
                <button id="next-to-step-3" class="px-8 py-3 bg-indigo-600 text-white font-bold rounded-md hover:bg-indigo-700 shadow-md">Valider les articles</button>
            </div>
        </div>

        <!-- Étape 3: Validation et Totaux -->
        <div id="step-3" class="wizard-step fade-in">
            <div class="bg-white p-8 rounded-xl shadow-lg max-w-5xl mx-auto">
                <h2 class="text-2xl font-bold text-gray-800 border-b pb-3 mb-6">Étape 3: Récapitulatif & Validation</h2>
                <div class="overflow-x-auto mb-6">
                    <table class="min-w-full divide-y divide-gray-200">
                        <thead id="order-table-head" class="bg-gray-50"></thead>
                        <tbody id="order-table-body" class="bg-white divide-y divide-gray-200"></tbody>
                    </table>
                </div>
                
                <section class="grid grid-cols-1 md:grid-cols-2 gap-8">
                    <!-- Options du devis -->
                    <div class="space-y-4">
                        <div id="discount-controls-container" class="hidden p-4 border rounded-lg">
                            <label class="block text-sm font-medium text-gray-700">Remise (%)</label>
                            <input type="number" id="clubDiscount" value="0" class="mt-1 block w-full md:w-1/2 rounded-md border-gray-300 shadow-sm" placeholder="0">
                        </div>
                         <div>
                            <label class="block text-sm font-medium text-gray-700">Options du devis</label>
                            <div class="mt-2 space-y-2">
                                <label class="flex items-center"><input id="apply-discount-check" type="checkbox" class="h-4 w-4 rounded border-gray-300"><span class="ml-2">Appliquer une remise</span></label>
                                <label class="flex items-center"><input id="custom-creation-check" type="checkbox" class="h-4 w-4 rounded border-gray-300"><span class="ml-2">Forfait Création Personnalisée</span></label>
                            </div>
                        </div>
                    </div>
                    <!-- Totaux finaux -->
                    <div class="space-y-2 text-right bg-slate-50 p-4 rounded-lg">
                        <div class="flex justify-between text-md"><span class="font-medium text-gray-600">Sous-total HT:</span><span id="subtotal-ht" class="font-semibold text-gray-800">0.00€</span></div>
                        <div class="flex justify-between text-md"><span class="font-medium text-gray-600">Sous-total TTC:</span><span id="subtotal-ttc" class="font-semibold text-gray-800">0.00€</span></div>
                        <div class="flex justify-between text-md text-red-600"><span class="font-medium">Remise TTC (Info):</span><span id="discount-amount-ttc" class="font-semibold">-0.00€</span></div>
                        <div class="flex justify-between text-md"><span class="font-medium text-gray-600">Frais de port TTC:</span><span id="shipping-ttc" class="font-semibold text-gray-800">0.00€</span></div>
                        <div id="graphic-fee-container" class="hidden flex justify-between text-md"><span class="font-medium text-gray-600">Forfait Création TTC:</span><span id="graphic-fee-ttc" class="font-semibold text-gray-800">0.00€</span></div>
                        <div class="border-t pt-2 mt-2"></div>
                        <div class="flex justify-between text-2xl"><span class="font-bold text-gray-700">Total Général TTC:</span><span id="grand-total-ttc" class="font-extrabold text-indigo-600">0.00€</span></div>
                        <div id="down-payment-container" class="hidden"></div>
                    </div>
                </section>

                <div class="mt-8 pt-6 border-t flex flex-col sm:flex-row justify-between items-center gap-4">
                    <button id="back-to-step-2" class="px-6 py-2 bg-gray-200 text-gray-700 font-medium rounded-md hover:bg-gray-300 w-full sm:w-auto">Précédent</button>
                     <div class="flex flex-col sm:flex-row gap-4 w-full sm:w-auto">
                        <button id="save-order-btn" class="w-full sm:w-auto inline-flex justify-center items-center px-6 py-3 border border-transparent text-base font-medium rounded-md shadow-sm text-white bg-blue-600 hover:bg-blue-700">Sauvegarder le Devis</button>
                        <button id="validate-order-btn" class="w-full sm:w-auto inline-flex justify-center items-center px-8 py-3 border border-transparent text-base font-bold rounded-md shadow-sm text-white bg-green-600 hover:bg-green-700 disabled:bg-green-300">Générer le Devis PDF</button>
                    </div>
                </div>
            </div>
        </div>
    </div>
    
    <script type="module">
    // =================================================================================
    // --- DATA & CONFIG ---
    // =================================================================================
    const allAvailableProducts = [
        { name: 'MAILLOT CLASSIQUE HOMME CONFORT MC', category: 'CYCLISME', type: 'haut', subtype: 'Maillots Manches Courtes', pricingGroup: 'maillotClassiqueMC', pricingTiers: [ { quantity: 1, price: 86.10 }, { quantity: 2, price: 73.80 }, { quantity: 3, price: 61.50 }, { quantity: 5, price: 49.20 }, { quantity: 15, price: 46.74 }, { quantity: 25, price: 45.26 }, { quantity: 50, price: 44.28 }, { quantity: 80, price: 42.80 }, { quantity: 150, price: 41.82 } ] },
        { name: 'MAILLOT CLASSIQUE FEMME CONFORT MC', category: 'CYCLISME', type: 'haut', subtype: 'Maillots Manches Courtes', pricingGroup: 'maillotClassiqueMC', pricingTiers: [ { quantity: 1, price: 86.10 }, { quantity: 2, price: 73.80 }, { quantity: 3, price: 61.50 }, { quantity: 5, price: 49.20 }, { quantity: 15, price: 46.74 }, { quantity: 25, price: 45.26 }, { quantity: 50, price: 44.28 }, { quantity: 80, price: 42.80 }, { quantity: 150, price: 41.82 } ] },
        { name: 'MAILLOT MIXTE CONFORT SANS MANCHE', category: 'CYCLISME', type: 'haut', subtype: 'Maillots Manches Courtes', pricingTiers: [ { quantity: 1, price: 86.10 }, { quantity: 2, price: 73.80 }, { quantity: 3, price: 61.50 }, { quantity: 5, price: 49.20 }, { quantity: 15, price: 46.74 }, { quantity: 25, price: 45.26 }, { quantity: 50, price: 44.28 }, { quantity: 80, price: 42.80 }, { quantity: 150, price: 41.82 } ] },
    { name: 'MAILLOT MIXTE PERFORMANCE MC', category: 'CYCLISME', type: 'haut', subtype: 'Maillots Manches Courtes', pricingGroup: 'maillotPerformanceMC', pricingTiers: [ { quantity: 1, price: 89.25 },{ quantity: 2, price: 76.50 }, { quantity: 3, price: 63.75 }, { quantity: 5, price: 51.00 }, { quantity: 15, price: 48.45 }, { quantity: 25, price: 46.92 }, { quantity: 50, price: 45.90 }, { quantity: 80, price: 44.37 }, { quantity: 150, price: 43.35 } ] },
        { name: 'MAILLOT MIXTE VTT CONFORT MC', category: 'CYCLISME', type: 'haut', subtype: 'Maillots Manches Courtes', pricingTiers: [ { quantity: 1, price: 89.25 }, { quantity: 2, price: 76.50 }, { quantity: 3, price: 63.75 }, { quantity: 5, price: 51.00 }, { quantity: 15, price: 48.45 }, { quantity: 25, price: 46.92 }, { quantity: 50, price: 45.90 }, { quantity: 80, price: 44.37 }, { quantity: 150, price: 43.35 } ] },
        { name: 'MAILLOT VTT/DESCENTE MIXTE CONFORT MC (Très ample)', category: 'CYCLISME', type: 'haut', subtype: 'Maillots Manches Courtes', sizeType: 'largeHaut', hasOptions: false, pricingTiers: [ { quantity: 1, price: 77.70 }, { quantity: 2, price: 66.60 }, { quantity: 3, price: 55.50 }, { quantity: 5, price: 44.40 }, { quantity: 15, price: 42.18 }, { quantity: 25, price: 40.85 }, { quantity: 50, price: 39.96 }, { quantity: 80, price: 38.63 }, { quantity: 150, price: 37.74 } ] },
        { name: 'MAILLOT MIXTE AERO MC', category: 'CYCLISME', type: 'haut', subtype: 'Maillots Manches Courtes', sizeType: 'aero', pricingTiers: [ { quantity: 1, price: 96.60 }, { quantity: 2, price: 82.80 }, { quantity: 3, price: 69.00 }, { quantity: 5, price: 55.20 }, { quantity: 15, price: 52.44 }, { quantity: 25, price: 50.78 }, { quantity: 50, price: 49.68 }, { quantity: 80, price: 48.02 }, { quantity: 150, price: 46.92 } ] },
        { name: 'MAILLOT MI-SAISON HOMME CONFORT ML', category: 'CYCLISME', type: 'haut', subtype: 'Maillots Manches Longues', pricingGroup: 'maillotMiSaisonML', pricingTiers: [ { quantity: 1, price: 101.85 }, { quantity: 2, price: 87.30 }, { quantity: 3, price: 72.75 }, { quantity: 5, price: 58.20 }, { quantity: 15, price: 55.29 }, { quantity: 25, price: 53.54 }, { quantity: 50, price: 52.38 }, { quantity: 80, price: 50.63 }, { quantity: 150, price: 49.47 } ] },
        { name: 'MAILLOT MI-SAISON FEMME CONFORT ML', category: 'CYCLISME', type: 'haut', subtype: 'Maillots Manches Longues', pricingGroup: 'maillotMiSaisonML', pricingTiers: [ { quantity: 1, price: 101.85 }, { quantity: 2, price: 87.30 }, { quantity: 3, price: 72.75 }, { quantity: 5, price: 58.20 }, { quantity: 15, price: 55.29 }, { quantity: 25, price: 53.54 }, { quantity: 50, price: 52.38 }, { quantity: 80, price: 50.63 }, { quantity: 150, price: 49.47 } ] },
        { name: 'MAILLOT BMX MIXTE CONFORT ML (Très ample)', category: 'CYCLISME', type: 'haut', subtype: 'Maillots Manches Longues', sizeType: 'largeHaut', hasOptions: false, pricingTiers: [ { quantity: 1, price: 88.20 }, { quantity: 2, price: 75.60 }, { quantity: 3, price: 63.00 }, { quantity: 5, price: 50.40 }, { quantity: 15, price: 47.88 }, { quantity: 25, price: 46.37 }, { quantity: 50, price: 45.36 }, { quantity: 80, price: 43.85 }, { quantity: 150, price: 42.84 } ] },
        { name: 'MAILLOT MI-SAISON MIXTE AERO ML', category: 'CYCLISME', type: 'haut', subtype: 'Maillots Manches Longues', sizeType: 'aero', pricingTiers: [ { quantity: 1, price: 107.10 }, { quantity: 2, price: 91.80 }, { quantity: 3, price: 76.50 }, { quantity: 5, price: 61.20 }, { quantity: 15, price: 58.14 }, { quantity: 25, price: 56.30 }, { quantity: 50, price: 55.08 }, { quantity: 80, price: 53.24 }, { quantity: 150, price: 52.02 } ] },
        { name: 'MAILLOT PLUIE MIXTE AERO MC (non sublimé, marquage DTF)', category: 'CYCLISME', type: 'haut', subtype: 'Essentiels et Vestes', pricingTiers: [ { quantity: 1, price: 149.10 }, { quantity: 2, price: 127.80 }, { quantity: 3, price: 106.50 }, { quantity: 5, price: 85.20 }, { quantity: 15, price: 80.94 }, { quantity: 25, price: 78.38 }, { quantity: 50, price: 76.68 }, { quantity: 80, price: 74.12 }, { quantity: 150, price: 72.42 } ] },
        { name: 'MAILLOT PLUIE MIXTE AERO ML (non sublimé, marquage DTF)', category: 'CYCLISME', type: 'haut', subtype: 'Essentiels et Vestes', pricingTiers: [ { quantity: 1, price: 178.50 }, { quantity: 2, price: 153.00 }, { quantity: 3, price: 127.50 }, { quantity: 5, price: 102.00 }, { quantity: 15, price: 96.90 }, { quantity: 25, price: 93.84 }, { quantity: 50, price: 91.80 }, { quantity: 80, price: 88.74 }, { quantity: 150, price: 86.70 } ] },
        { name: 'GILET COUPE-VENT LEGER MIXTE (vent et pluie fine, sans poche dos)', category: 'CYCLISME', type: 'haut', subtype: 'Essentiels et Vestes', pricingGroup: 'giletCoupeVent', excludedOptions: ['POCHE DOS ZIPPEE'], pricingTiers: [ { quantity: 1, price: 79.80 }, { quantity: 2, price: 68.40 }, { quantity: 3, price: 57.00 }, { quantity: 5, price: 45.60 }, { quantity: 15, price: 43.32 }, { quantity: 25, price: 41.95 }, { quantity: 50, price: 41.04 }, { quantity: 80, price: 39.67 }, { quantity: 150, price: 38.76 } ] },
        { name: 'GILET COUPE-VENT MI-SAISON MIXTE (dos ajouré)', category: 'CYCLISME', type: 'haut', subtype: 'Essentiels et Vestes', pricingGroup: 'giletCoupeVent', excludedOptions: ['POCHE DOS ZIPPEE'], pricingTiers: [ { quantity: 1, price: 98.70 }, { quantity: 2, price: 84.60 }, { quantity: 3, price: 70.50 }, { quantity: 5, price: 56.40 }, { quantity: 15, price: 53.58 }, { quantity: 25, price: 51.89 }, { quantity: 50, price: 50.76 }, { quantity: 80, price: 49.07 }, { quantity: 150, price: 47.94 } ] },
        { name: 'GILET COUPE-VENT HIVER MIXTE (tout membranné)', category: 'CYCLISME', type: 'haut', subtype: 'Essentiels et Vestes', pricingGroup: 'giletCoupeVent', excludedOptions: ['POCHE DOS ZIPPEE'], pricingTiers: [ { quantity: 1, price: 105.00 }, { quantity: 2, price: 90.00 }, { quantity: 3, price: 75.00 }, { quantity: 5, price: 60.00 }, { quantity: 15, price: 57.00 }, { quantity: 25, price: 55.20 }, { quantity: 50, price: 54.00 }, { quantity: 80, price: 52.20 }, { quantity: 150, price: 51.00 } ] },
        { name: 'COUPE-VENT LEGER MIXTE CONFORT (vent et pluie fine)', category: 'CYCLISME', type: 'haut', subtype: 'Essentiels et Vestes', pricingGroup: 'coupeVent', pricingTiers: [ { quantity: 1, price: 96.60 }, { quantity: 2, price: 82.80 }, { quantity: 3, price: 69.00 }, { quantity: 5, price: 55.20 }, { quantity: 15, price: 52.44 }, { quantity: 25, price: 50.78 }, { quantity: 50, price: 49.68 }, { quantity: 80, price: 48.02 }, { quantity: 150, price: 46.92 } ] },
        { name: 'COUPE-VENT LEGER DEPERLANT MIXTE CONFORT (avec membranne)', category: 'CYCLISME', type: 'haut', subtype: 'Essentiels et Vestes', pricingGroup: 'coupeVent', pricingTiers: [ { quantity: 1, price: 128.10 }, { quantity: 2, price: 109.80 }, { quantity: 3, price: 91.50 }, { quantity: 5, price: 73.20 }, { quantity: 15, price: 69.54 }, { quantity: 25, price: 67.34 }, { quantity: 50, price: 65.88 }, { quantity: 80, price: 63.68 }, { quantity: 150, price: 62.22 } ] },
        { name: 'VESTE MI-SAISON MIXTE CONFORT (membranne coupe-vent + mi-saison)', category: 'CYCLISME', type: 'haut', subtype: 'Essentiels et Vestes', pricingGroup: 'vesteMiSaison', pricingTiers: [ { quantity: 1, price: 136.50 }, { quantity: 2, price: 117.00 }, { quantity: 3, price: 97.50 }, { quantity: 5, price: 78.00 }, { quantity: 15, price: 74.10 }, { quantity: 25, price: 71.76 }, { quantity: 50, price: 70.20 }, { quantity: 80, price: 67.86 }, { quantity: 150, price: 66.30 } ] },
        { name: 'VESTE MI-SAISON MIXTE CONFORT avec -6cm aux ML', category: 'CYCLISME', type: 'haut', subtype: 'Essentiels et Vestes', pricingGroup: 'vesteMiSaison', pricingTiers: [ { quantity: 1, price: 136.50 }, { quantity: 2, price: 117.00 }, { quantity: 3, price: 97.50 }, { quantity: 5, price: 78.00 }, { quantity: 15, price: 74.10 }, { quantity: 25, price: 71.76 }, { quantity: 50, price: 70.20 }, { quantity: 80, price: 67.86 }, { quantity: 150, price: 66.30 } ] },
        { name: 'VESTE HIVER HOMME CONFORT', category: 'CYCLISME', type: 'haut', subtype: 'Essentiels et Vestes', pricingGroup: 'vesteHiverConfort', pricingTiers: [ { quantity: 1, price: 163.80 }, { quantity: 2, price: 140.40 }, { quantity: 3, price: 117.00 }, { quantity: 5, price: 93.60 }, { quantity: 15, price: 88.92 }, { quantity: 25, price: 86.11 }, { quantity: 50, price: 84.24 }, { quantity: 80, price: 81.43 }, { quantity: 150, price: 79.56 } ] },
        { name: 'VESTE HIVER FEMME CONFORT', category: 'CYCLISME', type: 'haut', subtype: 'Essentiels et Vestes', pricingGroup: 'vesteHiverConfort', pricingTiers: [ { quantity: 1, price: 163.80 }, { quantity: 2, price: 140.40 }, { quantity: 3, price: 117.00 }, { quantity: 5, price: 93.60 }, { quantity: 15, price: 88.92 }, { quantity: 25, price: 86.11 }, { quantity: 50, price: 84.24 }, { quantity: 80, price: 81.43 }, { quantity: 150, price: 79.56 } ] },
        { name: 'VESTE HIVER THERMIQUE HOMME CONFORT', category: 'CYCLISME', type: 'haut', subtype: 'Essentiels et Vestes', pricingGroup: 'vesteHiverThermique', pricingTiers: [ { quantity: 1, price: 168.00 }, { quantity: 2, price: 144.00 }, { quantity: 3, price: 120.00 }, { quantity: 5, price: 96.00 }, { quantity: 15, price: 91.20 }, { quantity: 25, price: 88.32 }, { quantity: 50, price: 86.40 }, { quantity: 80, price: 83.52 }, { quantity: 150, price: 81.60 } ] },
        { name: 'VESTE HIVER THERMIQUE FEMME CONFORT', category: 'CYCLISME', type: 'haut', subtype: 'Essentiels et Vestes', pricingGroup: 'vesteHiverThermique', pricingTiers: [ { quantity: 1, price: 168.00 }, { quantity: 2, price: 144.00 }, { quantity: 3, price: 120.00 }, { quantity: 5, price: 96.00 }, { quantity: 15, price: 91.20 }, { quantity: 25, price: 88.32 }, { quantity: 50, price: 86.40 }, { quantity: 80, price: 83.52 }, { quantity: 150, price: 81.60 } ] },
        { name: 'CUISSARD A BRETELLES HOMME CONFORT Peau LANDSCAPE', category: 'CYCLISME', type: 'haut', subtype: 'Cuissards Courts', isCuissardOrCollant: true, pricingGroup: 'cuissardConfortLandscape', pricingTiers: [ { quantity: 1, price: 121.80 }, { quantity: 2, price: 104.40 }, { quantity: 3, price: 87.00 }, { quantity: 5, price: 69.60 }, { quantity: 15, price: 66.12 }, { quantity: 25, price: 64.03 }, { quantity: 50, price: 62.64 }, { quantity: 80, price: 60.55 }, { quantity: 150, price: 59.16 }, ] },
        { name: 'CUISSARD A BRETELLES FEMME CONFORT Peau LANDSCAPE', category: 'CYCLISME', type: 'haut', subtype: 'Cuissards Courts', isCuissardOrCollant: true, pricingGroup: 'cuissardConfortLandscape', pricingTiers: [ { quantity: 1, price: 121.80 }, { quantity: 2, price: 104.40 }, { quantity: 3, price: 87.00 }, { quantity: 5, price: 69.60 }, { quantity: 15, price: 66.12 }, { quantity: 25, price: 64.03 }, { quantity: 50, price: 62.64 }, { quantity: 80, price: 60.55 }, { quantity: 150, price: 59.16 }, ] },
        { name: 'CUISSARD FEMME SANS BRETELLES CONFORT Peau LANDSCAPE', category: 'CYCLISME', type: 'haut', subtype: 'Cuissards Courts', isCuissardOrCollant: true, pricingGroup: 'cuissardConfortLandscape', pricingTiers: [ { quantity: 1, price: 117.60 }, { quantity: 2, price: 100.80 }, { quantity: 3, price: 84.00 }, { quantity: 5, price: 67.20 }, { quantity: 15, price: 63.84 }, { quantity: 25, price: 61.82 }, { quantity: 50, price: 60.48 }, { quantity: 80, price: 58.46 }, { quantity: 150, price: 57.12 }, ] },
        { name: 'CUISSARD HOMME AERO Peau CERVINO', category: 'CYCLISME', type: 'haut', subtype: 'Cuissards Courts', sizeType: 'aero', isCuissardOrCollant: true, pricingGroup: 'cuissardAeroCervino', pricingTiers: [ { quantity: 1, price: 142.80 }, { quantity: 2, price: 122.40 }, { quantity: 3, price: 102.00 }, { quantity: 5, price: 81.60 }, { quantity: 15, price: 77.52 }, { quantity: 25, price: 75.07 }, { quantity: 50, price: 73.44 }, { quantity: 80, price: 70.99 }, { quantity: 150, price: 69.36 }, ] },
        { name: 'CUISSARD FEMME AERO Peau CERVINO', category: 'CYCLISME', type: 'haut', subtype: 'Cuissards Courts', sizeType: 'aero', isCuissardOrCollant: true, pricingGroup: 'cuissardAeroCervino', pricingTiers: [ { quantity: 1, price: 142.80 }, { quantity: 2, price: 122.40 }, { quantity: 3, price: 102.00 }, { quantity: 5, price: 81.60 }, { quantity: 15, price: 77.52 }, { quantity: 25, price: 75.07 }, { quantity: 50, price: 73.44 }, { quantity: 80, price: 70.99 }, { quantity: 150, price: 69.36 }, ] },
        { name: 'SHORT VTT FOND Peau ENDURANCE 2.5', category: 'CYCLISME', type: 'haut', subtype: 'Cuissards Courts', isCuissardOrCollant: true, pricingTiers: [ { quantity: 1, price: 75.00 } ] },
        { name: 'CORSAIRE HOMME A BRETELLES CONFORT', category: 'CYCLISME', type: 'haut', subtype: 'Corsaires/Collants', sizeType: 'ample', isCuissardOrCollant: true, pricingGroup: 'corsaireConfortLandscape', pricingTiers: [ { quantity: 1, price: 98.70 } ] },
        { name: 'CORSAIRE FEMME SANS BRETELLES CONFORT', category: 'CYCLISME', type: 'haut', subtype: 'Corsaires/Collants', sizeType: 'ample', isCuissardOrCollant: true, pricingGroup: 'corsaireConfortLandscape', pricingTiers: [ { quantity: 1, price: 95.70 } ] },
        { name: 'COLLANT HIVER A BRETELLES HOMME CONFORT', category: 'CYCLISME', type: 'haut', subtype: 'Corsaires/Collants', sizeType: 'ample', isCuissardOrCollant: true, pricingGroup: 'collantHiverConfortLandscape', pricingTiers: [ { quantity: 1, price: 138.60 }, { quantity: 2, price: 118.80 }, { quantity: 3, price: 99.00 }, { quantity: 5, price: 79.20 }, { quantity: 15, price: 75.24 }, { quantity: 25, price: 72.86 }, { quantity: 50, price: 71.28 }, { quantity: 80, price: 68.90 }, { quantity: 150, price: 67.32 }, ] },
        { name: 'COLLANT HIVER A BRETELLES FEMME CONFORT', category: 'CYCLISME', type: 'haut', subtype: 'Corsaires/Collants', sizeType: 'ample', isCuissardOrCollant: true, pricingGroup: 'collantHiverConfortLandscape', pricingTiers: [ { quantity: 1, price: 138.60 }, { quantity: 2, price: 118.80 }, { quantity: 3, price: 99.00 }, { quantity: 5, price: 79.20 }, { quantity: 15, price: 75.24 }, { quantity: 25, price: 72.86 }, { quantity: 50, price: 71.28 }, { quantity: 80, price: 68.90 }, { quantity: 150, price: 67.32 }, ] },
        { name: 'COLLANT HIVER FEMME SANS BRETELLES CONFORT', category: 'CYCLISME', type: 'haut', subtype: 'Corsaires/Collants', sizeType: 'ample', isCuissardOrCollant: true, pricingGroup: 'collantHiverConfortLandscape', pricingTiers: [ { quantity: 1, price: 134.40 }, { quantity: 2, price: 115.20 }, { quantity: 3, price: 96.00 }, { quantity: 5, price: 76.80 }, { quantity: 15, price: 72.96 }, { quantity: 25, price: 70.66 }, { quantity: 50, price: 69.12 }, { quantity: 80, price: 66.82 }, { quantity: 150, price: 65.28 }, ] },
        { name: 'COLLANT HIVER HOMME AERO Peau CERVINO', category: 'CYCLISME', type: 'haut', subtype: 'Corsaires/Collants', pricingGroup: 'collantHiverAeroCervino', sizeType: 'aero', isCuissardOrCollant: true, pricingTiers: [ { quantity: 1, price: 165.90 }, { quantity: 2, price: 142.20 }, { quantity: 3, price: 118.50 }, { quantity: 5, price: 94.80 }, { quantity: 15, price: 90.06 }, { quantity: 25, price: 87.22 }, { quantity: 50, price: 85.32 }, { quantity: 80, price: 82.48 }, { quantity: 150, price: 80.58 }, ] },
        { name: 'COLLANT HIVER FEMME AERO Peau CERVINO', category: 'CYCLISME', type: 'haut', subtype: 'Corsaires/Collants', pricingGroup: 'collantHiverAeroCervino', sizeType: 'aero', isCuissardOrCollant: true, pricingTiers: [ { quantity: 1, price: 165.90 }, { quantity: 2, price: 142.20 }, { quantity: 3, price: 118.50 }, { quantity: 5, price: 94.80 }, { quantity: 15, price: 90.06 }, { quantity: 25, price: 87.22 }, { quantity: 50, price: 85.32 }, { quantity: 80, price: 82.48 }, { quantity: 150, price: 80.58 }, ] },
        { name: 'COLLANT MIXTE ECHAUFFEMENT', category: 'CYCLISME', type: 'haut', subtype: 'Corsaires/Collants', sizeType: 'ample', isCuissardOrCollant: true, pricingTiers: [ { quantity: 1, price: 98.70 }, { quantity: 2, price: 84.60 }, { quantity: 3, price: 70.50 }, { quantity: 5, price: 56.40 }, { quantity: 15, price: 53.58 }, { quantity: 25, price: 51.89 }, { quantity: 50, price: 50.76 }, { quantity: 80, price: 49.07 }, { quantity: 150, price: 47.94 }, ] },
        { name: 'COMBINAISON ROUTE MANCHES COURTES HOMME AERO', category: 'CYCLISME', type: 'haut', subtype: 'Combinaisons', sizeType: 'aero', pricingTiers: [{ quantity: 1, price: 115.20 }] },
        { name: 'COMBINAISON ROUTE MANCHES COURTES FEMME AERO', category: 'CYCLISME', type: 'haut', subtype: 'Combinaisons', sizeType: 'aero', pricingTiers: [{ quantity: 1, price: 115.20 }] },
        { name: 'COMBINAISON CHRONO ROUTE MANCHES COURTES HOMME AERO', category: 'CYCLISME', type: 'haut', subtype: 'Combinaisons', sizeType: 'aero', pricingTiers: [{ quantity: 1, price: 115.20 }] },
        { name: 'COMBINAISON CHRONO ROUTE MANCHES COURTES FEMME AERO', category: 'CYCLISME', type: 'haut', subtype: 'Combinaisons', sizeType: 'aero', pricingTiers: [{ quantity: 1, price: 115.20 }] },
        { name: 'COMBINAISON CHRONO ROUTE MANCHES LONGUES HOMME AERO', category: 'CYCLISME', type: 'haut', subtype: 'Combinaisons', sizeType: 'aero', pricingTiers: [{ quantity: 1, price: 120.00 }] },
        { name: 'COMBINAISON CHRONO ROUTE MANCHES LONGUES FEMME AERO', category: 'CYCLISME', type: 'haut', subtype: 'Combinaisons', sizeType: 'aero', pricingTiers: [{ quantity: 1, price: 120.00 }] },
        { name: 'COMBINAISON CHRONO PISTE MANCHES COURTES HOMME AERO', category: 'CYCLISME', type: 'haut', subtype: 'Combinaisons', sizeType: 'aero', pricingTiers: [{ quantity: 1, price: 111.60 }] },
        { name: 'COMBINAISON CHRONO PISTE MANCHES COURTES FEMME AERO', category: 'CYCLISME', type: 'haut', subtype: 'Combinaisons', sizeType: 'aero', pricingTiers: [{ quantity: 1, price: 111.60 }] },
        { name: 'COMBINAISON CHRONO PISTE MANCHES LONGUES HOMME AERO', category: 'CYCLISME', type: 'haut', subtype: 'Combinaisons', sizeType: 'aero', pricingTiers: [{ quantity: 1, price: 116.40 }] },
        { name: 'COMBINAISON CHRONO PISTE MANCHES LONGUES FEMME AERO', category: 'CYCLISME', type: 'haut', subtype: 'Combinaisons', sizeType: 'aero', pricingTiers: [{ quantity: 1, price: 111.60 }] },
        { name: 'COMBINAISON CYCLO-CROSS MANCHES LONGUES HOMME AERO', category: 'CYCLISME', type: 'haut', subtype: 'Combinaisons', sizeType: 'aero', pricingTiers: [{ quantity: 1, price: 120.00 }] },
        { name: 'COMBINAISON CYCLO-CROSSMANCHES LONGUES FEMME AERO', category: 'CYCLISME', type: 'haut', subtype: 'Combinaisons', sizeType: 'aero', pricingTiers: [{ quantity: 1, price: 120.00 }] },
        { name: 'MAILLOT RUNNING HOMME', category: 'RUNNING', type: 'haut', subtype: 'Hauts', pricingGroup: 'maillotRunning', excludedOptions: ['POCHE DOS ZIPPEE'], pricingTiers: [ { quantity: 1, price: 63.00 }, { quantity: 2, price: 54.00 }, { quantity: 3, price: 45.00 }, { quantity: 5, price: 36.00 }, { quantity: 15, price: 34.20 }, { quantity: 25, price: 33.12 }, { quantity: 50, price: 32.40 }, { quantity: 80, price: 31.32 }, { quantity: 150, price: 30.60 } ] },
        { name: 'MAILLOT RUNNING FEMME', category: 'RUNNING', type: 'haut', subtype: 'Hauts', pricingGroup: 'maillotRunning', excludedOptions: ['POCHE DOS ZIPPEE'], pricingTiers: [ { quantity: 1, price: 63.00 }, { quantity: 2, price: 54.00 }, { quantity: 3, price: 45.00 }, { quantity: 5, price: 36.00 }, { quantity: 15, price: 34.20 }, { quantity: 25, price: 33.12 }, { quantity: 50, price: 32.40 }, { quantity: 80, price: 31.32 }, { quantity: 150, price: 30.60 } ] },
        { name: 'MAILLOT TRAIL HOMME MANCHES COURTES', category: 'RUNNING', type: 'haut', subtype: 'Hauts', pricingGroup: 'maillotTrail', excludedOptions: ['POCHE DOS ZIPPEE'], pricingTiers: [ { quantity: 1, price: 87.15 }, { quantity: 2, price: 74.70 }, { quantity: 3, price: 62.25 }, { quantity: 5, price: 49.80 }, { quantity: 15, price: 47.31 }, { quantity: 25, price: 45.82 }, { quantity: 50, price: 44.82 }, { quantity: 80, price: 43.33 }, { quantity: 150, price: 42.33 } ] },
        { name: 'MAILLOT TRAIL FEMME MANCHES COURTES', category: 'RUNNING', type: 'haut', subtype: 'Hauts', pricingGroup: 'maillotTrail', excludedOptions: ['POCHE DOS ZIPPEE'], pricingTiers: [ { quantity: 1, price: 87.15 }, { quantity: 2, price: 74.70 }, { quantity: 3, price: 62.25 }, { quantity: 5, price: 49.80 }, { quantity: 15, price: 47.31 }, { quantity: 25, price: 45.82 }, { quantity: 50, price: 44.82 }, { quantity: 80, price: 43.33 }, { quantity: 150, price: 42.33 } ] },
        { name: 'DEBARDEUR ATHLETISME HOMME', category: 'RUNNING', type: 'haut', subtype: 'Hauts', pricingGroup: 'debardeurAthletisme', excludedOptions: ['POCHE DOS ZIPPEE'], pricingTiers: [ { quantity: 1, price: 56.70 }, { quantity: 2, price: 48.60 }, { quantity: 3, price: 40.50 }, { quantity: 5, price: 32.40 }, { quantity: 15, price: 30.78 }, { quantity: 25, price: 29.81 }, { quantity: 50, price: 29.16 }, { quantity: 80, price: 28.19 }, { quantity: 150, price: 27.54 } ] },
        { name: 'DEBARDEUR ATHLETISME FEMME', category: 'RUNNING', type: 'haut', subtype: 'Hauts', pricingGroup: 'debardeurAthletisme', excludedOptions: ['POCHE DOS ZIPPEE'], pricingTiers: [ { quantity: 1, price: 56.70 }, { quantity: 2, price: 48.60 }, { quantity: 3, price: 40.50 }, { quantity: 5, price: 32.40 }, { quantity: 15, price: 30.78 }, { quantity: 25, price: 29.81 }, { quantity: 50, price: 29.16 }, { quantity: 80, price: 28.19 }, { quantity: 150, price: 27.54 } ] },
        { name: 'BRASSIERE RUNNING FEMME', category: 'RUNNING', type: 'haut', subtype: 'Hauts', hasOptions: false, pricingTiers: [ { quantity: 1, price: 68.25 }, { quantity: 2, price: 58.50 }, { quantity: 3, price: 48.75 }, { quantity: 5, price: 39.00 }, { quantity: 15, price: 37.05 }, { quantity: 25, price: 35.88 }, { quantity: 50, price: 35.10 }, { quantity: 80, price: 33.93 }, { quantity: 150, price: 33.15 } ] },
        { name: 'MAILLOT RUNNING HIVER HOMME MANCHES LONGUES', category: 'RUNNING', type: 'haut', subtype: 'Hauts', pricingGroup: 'maillotRunningHiver', excludedOptions: ['POCHE DOS ZIPPEE'], pricingTiers: [ { quantity: 1, price: 94.50 }, { quantity: 2, price: 81.00 }, { quantity: 3, price: 67.50 }, { quantity: 5, price: 54.00 }, { quantity: 15, price: 51.30 }, { quantity: 25, price: 49.68 }, { quantity: 50, price: 48.60 }, { quantity: 80, price: 46.98 }, { quantity: 150, price: 45.90 } ] },
        { name: 'MAILLOT RUNNING HIVER FEMME MANCHES LONGUES', category: 'RUNNING', type: 'haut', subtype: 'Hauts', pricingGroup: 'maillotRunningHiver', excludedOptions: ['POCHE DOS ZIPPEE'], pricingTiers: [ { quantity: 1, price: 94.50 }, { quantity: 2, price: 81.00 }, { quantity: 3, price: 67.50 }, { quantity: 5, price: 54.00 }, { quantity: 15, price: 51.30 }, { quantity: 25, price: 49.68 }, { quantity: 50, price: 48.60 }, { quantity: 80, price: 46.98 }, { quantity: 150, price: 45.90 } ] },
        { name: 'GILET COUPE-VENT LEGER MIXTE', category: 'RUNNING', type: 'haut', subtype: 'Hauts', excludedOptions: ['POCHE DOS ZIPPEE'], pricingTiers: [ { quantity: 1, price: 79.80 }, { quantity: 2, price: 68.40 }, { quantity: 3, price: 57.00 }, { quantity: 5, price: 45.60 }, { quantity: 15, price: 43.32 }, { quantity: 25, price: 41.95 }, { quantity: 50, price: 41.04 }, { quantity: 80, price: 39.67 }, { quantity: 150, price: 38.76 } ] },
        { name: 'GILET COUPE-VENT MI-SAISON MIXTE', category: 'RUNNING', type: 'haut', subtype: 'Hauts', excludedOptions: ['POCHE DOS ZIPPEE'], pricingTiers: [ { quantity: 1, price: 98.70 }, { quantity: 2, price: 84.60 }, { quantity: 3, price: 70.50 }, { quantity: 5, price: 56.40 }, { quantity: 15, price: 53.58 }, { quantity: 25, price: 51.89 }, { quantity: 50, price: 50.76 }, { quantity: 80, price: 49.07 }, { quantity: 150, price: 47.94 } ] },
        { name: 'GILET COUPE-VENT HIVER MIXTE', category: 'RUNNING', type: 'haut', subtype: 'Hauts', excludedOptions: ['POCHE DOS ZIPPEE'], pricingTiers: [ { quantity: 1, price: 105.00 }, { quantity: 2, price: 90.00 }, { quantity: 3, price: 75.00 }, { quantity: 5, price: 60.00 }, { quantity: 15, price: 57.00 }, { quantity: 25, price: 55.20 }, { quantity: 50, price: 54.00 }, { quantity: 80, price: 52.20 }, { quantity: 150, price: 51.00 } ] },
        { name: 'COUPE-VENT LEGER MIXTE CONFORT', category: 'RUNNING', type: 'haut', subtype: 'Hauts', excludedOptions: ['POCHE DOS ZIPPEE'], pricingTiers: [ { quantity: 1, price: 96.60 }, { quantity: 2, price: 82.80 }, { quantity: 3, price: 69.00 }, { quantity: 5, price: 55.20 }, { quantity: 15, price: 52.44 }, { quantity: 25, price: 50.78 }, { quantity: 50, price: 49.68 }, { quantity: 80, price: 48.02 }, { quantity: 150, price: 46.92 } ] },
        { name: 'VESTE MI-SAISON HOMME CONFORT', category: 'RUNNING', type: 'haut', subtype: 'Hauts', pricingTiers: [ { quantity: 1, price: 136.50 }, { quantity: 2, price: 117.00 }, { quantity: 3, price: 97.50 }, { quantity: 5, price: 78.00 }, { quantity: 15, price: 74.10 }, { quantity: 25, price: 71.76 }, { quantity: 50, price: 70.20 }, { quantity: 80, price: 67.86 }, { quantity: 150, price: 66.30 } ] },
        { name: 'VESTE MI-SAISON FEMME CONFORT', category: 'RUNNING', type: 'haut', subtype: 'Hauts', pricingTiers: [ { quantity: 1, price: 136.50 }, { quantity: 2, price: 117.00 }, { quantity: 3, price: 97.50 }, { quantity: 5, price: 78.00 }, { quantity: 15, price: 74.10 }, { quantity: 25, price: 71.76 }, { quantity: 50, price: 70.20 }, { quantity: 80, price: 67.86 }, { quantity: 150, price: 66.30 } ] },
        { name: 'SHORT RUNNING MIXTE', category: 'RUNNING', type: 'haut', subtype: 'Bas', excludedOptions: ['POCHE DOS ZIPPEE'], pricingTiers: [ { quantity: 1, price: 79.80 }, { quantity: 2, price: 68.40 }, { quantity: 3, price: 57.00 }, { quantity: 5, price: 45.60 }, { quantity: 15, price: 43.32 }, { quantity: 25, price: 41.95 }, { quantity: 50, price: 41.04 }, { quantity: 80, price: 39.67 }, { quantity: 150, price: 38.76 } ] },
        { name: 'SHORTY FEMME RUNNING', category: 'RUNNING', type: 'haut', subtype: 'Bas', pricingGroup: 'cuissardShortyRunning', excludedOptions: ['POCHE DOS ZIPPEE'], pricingTiers: [ { quantity: 1, price: 84.00 }, { quantity: 2, price: 72.00 }, { quantity: 3, price: 60.00 }, { quantity: 5, price: 48.00 }, { quantity: 15, price: 45.60 }, { quantity: 25, price: 44.16 }, { quantity: 50, price: 43.20 }, { quantity: 80, price: 41.76 }, { quantity: 150, price: 40.80 } ] },
        { name: 'CUISSARD RUNNING HOMME', category: 'RUNNING', type: 'haut', subtype: 'Bas', pricingGroup: 'cuissardShortyRunning', excludedOptions: ['POCHE DOS ZIPPEE'], pricingTiers: [ { quantity: 1, price: 84.00 }, { quantity: 2, price: 72.00 }, { quantity: 3, price: 60.00 }, { quantity: 5, price: 48.00 }, { quantity: 15, price: 45.60 }, { quantity: 25, price: 44.16 }, { quantity: 50, price: 43.20 }, { quantity: 80, price: 41.76 }, { quantity: 150, price: 40.80 } ] },
        { name: 'COLLANT RUNNING MIXTE', category: 'RUNNING', type: 'haut', subtype: 'Bas', excludedOptions: ['POCHE DOS ZIPPEE'], pricingTiers: [ { quantity: 1, price: 105.00 }, { quantity: 2, price: 90.00 }, { quantity: 3, price: 75.00 }, { quantity: 5, price: 60.00 }, { quantity: 15, price: 57.00 }, { quantity: 25, price: 55.20 }, { quantity: 50, price: 54.00 }, { quantity: 80, price: 52.20 }, { quantity: 150, price: 51.00 } ] },
        { name: 'TRIFONCTION HOMME COURTE DISTANCE Peau TRI GEL', category: 'RUNNING', type: 'haut', subtype: 'Trifonctions', hasOptions: false, pricingGroup: 'trifonctionCourte', pricingTiers: [ { quantity: 1, price: 134.40 }, { quantity: 2, price: 115.20 }, { quantity: 3, price: 96.00 }, { quantity: 5, price: 76.80 }, { quantity: 15, price: 72.96 }, { quantity: 25, price: 70.66 }, { quantity: 50, price: 69.12 }, { quantity: 80, price: 66.82 }, { quantity: 150, price: 65.28 } ] },
        { name: 'TRIFONCTION FEMME COURTE DISTANCE Peau TRI GEL', category: 'RUNNING', type: 'haut', subtype: 'Trifonctions', hasOptions: false, pricingGroup: 'trifonctionCourte', pricingTiers: [ { quantity: 1, price: 134.40 }, { quantity: 2, price: 115.20 }, { quantity: 3, price: 96.00 }, { quantity: 5, price: 76.80 }, { quantity: 15, price: 72.96 }, { quantity: 25, price: 70.66 }, { quantity: 50, price: 69.12 }, { quantity: 80, price: 66.82 }, { quantity: 150, price: 65.28 } ] },
        { name: 'TRIFONCTION HOMME HALF Peau TRI GEL, ZIP DEVANT OU DOS', category: 'RUNNING', type: 'haut', subtype: 'Trifonctions', hasOptions: false, pricingGroup: 'trifonctionHalf', pricingTiers: [ { quantity: 1, price: 184.80 }, { quantity: 2, price: 158.40 }, { quantity: 3, price: 132.00 }, { quantity: 5, price: 105.60 }, { quantity: 15, price: 100.32 }, { quantity: 25, price: 97.15 }, { quantity: 50, price: 95.04 }, { quantity: 80, price: 91.87 }, { quantity: 150, price: 89.76 } ] },
        { name: 'TRIFONCTION FEMME HALF Peau TRI GEL, ZIP DEVANT OU DOS', category: 'RUNNING', type: 'haut', subtype: 'Trifonctions', hasOptions: false, pricingGroup: 'trifonctionHalf', pricingTiers: [ { quantity: 1, price: 184.80 }, { quantity: 2, price: 158.40 }, { quantity: 3, price: 132.00 }, { quantity: 5, price: 105.60 }, { quantity: 15, price: 100.32 }, { quantity: 25, price: 97.15 }, { quantity: 50, price: 95.04 }, { quantity: 80, price: 91.87 }, { quantity: 150, price: 89.76 } ] },
        { name: 'BANDANA ÉTÉ', category: 'ACCESSOIRES', type: 'accessoire', subtype: 'ACCESSOIRES PERSONNALISÉS', sizeType:'unique', minQuantity: 10, pricingTiers: [ { quantity: 10, price: 12.00 }, { quantity: 20, price: 10.44 }, { quantity: 50, price: 10.20 } ] },
        { name: 'BANDEAU', category: 'ACCESSOIRES', type: 'accessoire', subtype: 'ACCESSOIRES PERSONNALISÉS', sizeType:'unique', minQuantity: 10, pricingTiers: [ { quantity: 10, price: 9.00 },  { quantity: 20, price: 8.40 }, { quantity: 50, price: 7.20 } ] },
        { name: 'TOUR DE COU', category: 'ACCESSOIRES', type: 'accessoire', subtype: 'ACCESSOIRES PERSONNALISÉS', sizeType:'unique', minQuantity: 10, pricingTiers: [ { quantity: 10, price: 10.20 }, { quantity: 20, price: 8.70 }, { quantity: 50, price: 8.40 } ] },
        { name: 'PASSE MONTAGNE', category: 'ACCESSOIRES', type: 'accessoire', subtype: 'ACCESSOIRES PERSONNALISÉS', sizeType:'unique', minQuantity: 10, pricingTiers: [ { quantity: 10, price: 18.00 }, { quantity: 20, price: 15.60 }, { quantity: 50, price: 14.40 } ] },
        { name: 'MANCHETTES HIVER VELO/RUNNING', category: 'ACCESSOIRES', type: 'accessoire', subtype: 'ACCESSOIRES PERSONNALISÉS', sizeType:'manchettes', minQuantity: 10, pricingTiers: [ { quantity: 10, price: 19.20 }, { quantity: 20, price: 16.80 }, { quantity: 50, price: 15.60 } ] },
        { name: 'JAMBIERES', category: 'ACCESSOIRES', type: 'accessoire', subtype: 'ACCESSOIRES PERSONNALISÉS', sizeType:'jambieres', minQuantity: 10, pricingTiers: [ { quantity: 10, price: 26.40 }, { quantity: 20, price: 24.00 }, { quantity: 50, price: 22.80 } ] },
        { name: 'GANTS ÉTÉ', category: 'ACCESSOIRES', type: 'accessoire', subtype: 'ACCESSOIRES PERSONNALISÉS', sizeType:'gants', minQuantity: 10, pricingTiers: [ { quantity: 10, price: 21.00 }, { quantity: 20, price: 18.00 }, { quantity: 50, price: 16.80 } ] },
        { name: 'TAPIS DE TRANSITION MULTISPORTS', category: 'ACCESSOIRES', type: 'accessoire', subtype: 'ACCESSOIRES PERSONNALISÉS', sizeType:'unique', minQuantity: 10, pricingTiers: [ { quantity: 10, price: 10.80 }, { quantity: 20, price: 9.18 }, { quantity: 50, price: 8.40 } ] },
        { name: 'CHAUSSETTES AERO MIXTE 18cm', category: 'ACCESSOIRES', type: 'accessoire', subtype: 'ACCESSOIRES PERSONNALISÉS', sizeType:'chaussettes', minQuantity: 10, pricingTiers: [ { quantity: 10, price: 21.00 }, { quantity: 20, price: 20.40 }, { quantity: 50, price: 19.20 } ] },
        { name: 'CHAUSSETTES VELO/COURSE A PIED Mixte Tige 13 ou 17cm', category: 'ACCESSOIRES', type: 'accessoire', subtype: 'ACCESSOIRES PERSONNALISÉS', sizeType:'chaussettes', minQuantity: 50, pricingTiers: [ { quantity: 50, price: 12.60 }, { quantity: 100, price: 12.00 }, { quantity: 200, price: 12.00 } ] },
        { name: 'GAPETTE VELO', category: 'ACCESSOIRES', type: 'accessoire', subtype: 'ACCESSOIRES PERSONNALISÉS', sizeType:'unique', minQuantity: 50, pricingTiers: [ { quantity: 50, price: 15.00 }, { quantity: 100, price: 13.20 }, { quantity: 200, price: 13.20 } ] },
        { name: 'DOSSARDS JEU DE 1 à 100', category: 'ACCESSOIRES', type: 'accessoire', subtype: 'ACCESSOIRES PERSONNALISÉS', sizeType:'unique', pricingTiers: [{quantity: 1, price: 68.80}] },
        { name: 'DOSSARDS JEU DE 1 à 150', category: 'ACCESSOIRES', type: 'accessoire', subtype: 'ACCESSOIRES PERSONNALISÉS', sizeType:'unique', pricingTiers: [{quantity: 1, price: 91.20}] },
        { name: 'DOSSARDS JEU DE 1 à 200', category: 'ACCESSOIRES', type: 'accessoire', subtype: 'ACCESSOIRES PERSONNALISÉS', sizeType:'unique', pricingTiers: [{quantity: 1, price: 115.20}] },
        { name: 'DOSSARDS JEU DE 1 à 250', category: 'ACCESSOIRES', type: 'accessoire', subtype: 'ACCESSOIRES PERSONNALISÉS', sizeType:'unique', pricingTiers: [{quantity: 1, price: 136.00}] },
        { name: 'DOSSARDS JEU DE 1 à 300', category: 'ACCESSOIRES', type: 'accessoire', subtype: 'ACCESSOIRES PERSONNALISÉS', sizeType:'unique', pricingTiers: [{quantity: 1, price: 158.40}] },
        { name: 'SOUS-MAILLOT SANS MANCHES', category: 'ACCESSOIRES', type: 'accessoire', subtype: 'ACCESSOIRES PERMANENTS', sizeType: 'sousMaillot', price: 40, colors: ["blanc"]},
        { name: 'SOUS-MAILLOT MI-SAISON MANCHES COURTES', category: 'ACCESSOIRES', type: 'accessoire', subtype: 'ACCESSOIRES PERMANENTS', sizeType: 'sousMaillot', price: 45, colors: ["blanc"]},
        { name: 'SOUS-MAILLOT HIVER MANCHES LONGUES', category: 'ACCESSOIRES', type: 'accessoire', subtype: 'ACCESSOIRES PERMANENTS', sizeType: 'sousMaillotHiver', price: 55, colors: ["blanc"]},
        { name: 'SOUS CASQUE', category: 'ACCESSOIRES', type: 'accessoire', subtype: 'ACCESSOIRES PERMANENTS', sizeType: 'unique', price: 18, colors: ["NOIR"]},
        { name: 'CAGOULE', category: 'ACCESSOIRES', type: 'accessoire', subtype: 'ACCESSOIRES PERMANENTS', sizeType: 'unique', price: 20, colors: ["NOIR"]},
        { name: 'GANTS HIVER', category: 'ACCESSOIRES', type: 'accessoire', subtype: 'ACCESSOIRES PERMANENTS', sizeType: 'gants', price: 55, colors: ["NOIR"]},
        { name: 'GANTS ETE CONFORT', category: 'ACCESSOIRES', type: 'accessoire', subtype: 'ACCESSOIRES PERMANENTS', sizeType: 'gants', price: 30, colors: ["NOIR", "BLANC", "MARINE", "BRETON PUR BEURRE"]},
        { name: 'GANTS ETE SLIM', category: 'ACCESSOIRES', type: 'accessoire', subtype: 'ACCESSOIRES PERMANENTS', sizeType: 'gants', price: 40, colors: ["NOIR"]},
        { name: 'GANTS MI-SAISON', category: 'ACCESSOIRES', type: 'accessoire', subtype: 'ACCESSOIRES PERMANENTS', sizeType: 'gantsMiSaison', price: 30, colors: ["NOIR"]},
        { name: 'COUVRE-CHAUSSURES AÉRO', category: 'ACCESSOIRES', type: 'accessoire', subtype: 'ACCESSOIRES PERMANENTS', sizeType: 'couvreChaussuresAero', price: 40, colors: ["NOIR"]},
        { name: 'COUVRE-CHAUSSURES HIVER/PLUIE', category: 'ACCESSOIRES', type: 'accessoire', subtype: 'ACCESSOIRES PERMANENTS', sizeType: 'couvreChaussuresHiver', price: 65, colors: ["NOIR"]},
        { name: 'BANDEAU', category: 'ACCESSOIRES', type: 'accessoire', subtype: 'ACCESSOIRES PERMANENTS', sizeType: 'bandeau', price: 12, colors: ["ARDENT", "FLUO", "HYPNOTIC", "BRETON PUR BEURRE"]},
        { name: 'TOUR DE COU', category: 'ACCESSOIRES', type: 'accessoire', subtype: 'ACCESSOIRES PERMANENTS', sizeType: 'tourDeCou', price: 15, colors: ["ARDENT", "FLUO", "HYPNOTIC", "BRETON PUR BEURRE"]},
        { name: 'GAPETTE', category: 'ACCESSOIRES', type: 'accessoire', subtype: 'ACCESSOIRES PERMANENTS', sizeType: 'unique', price: 20, colors: ["ARDENT", "MAGICREME", "HYPNOTIC", "NOIR"]},
        { name: 'MANCHETTES', category: 'ACCESSOIRES', type: 'accessoire', subtype: 'ACCESSOIRES PERMANENTS', sizeType: 'manchettes', price: 33, colors: ["NOIR"]},
        { name: 'GENOUILLERES', category: 'ACCESSOIRES', type: 'accessoire', subtype: 'ACCESSOIRES PERMANENTS', sizeType: 'manchettes', price: 33, colors: ["NOIR"]},
        { name: 'JAMBIERES', category: 'ACCESSOIRES', type: 'accessoire', subtype: 'ACCESSOIRES PERMANENTS', sizeType: 'jambieres', price: 40, colors: ["NOIR"]},
        { name: 'CHAUSSETTES AÉRO', category: 'ACCESSOIRES', type: 'accessoire', subtype: 'ACCESSOIRES PERMANENTS', sizeType: 'chaussettes', price: 30, colors: ["NOIR", "BLANC", "ARDENT", "HYPNOTIC"]},
        { name: 'CHAUSSETTES MI-HAUTES', category: 'ACCESSOIRES', type: 'accessoire', subtype: 'ACCESSOIRES PERMANENTS', sizeType: 'chaussettes', price: 17, colors: ["NOIR", "BLANC", "BEIGE", "BRETON PUR BEURRE"]},
        { name: 'MAILLOT ENFANT PERFORMANCE MC', category: 'ENFANTS', type: 'enfant', subtype: 'Cyclisme Enfant', hasOptions: false, pricingTiers: [{ quantity: 1, price: 42.00 }] },
        { name: 'MAILLOT VTT/DESCENTE ENFANT CONFORT MC', category: 'ENFANTS', type: 'enfant', subtype: 'Cyclisme Enfant', hasOptions: false, pricingTiers: [{ quantity: 1, price: 44.40 }] },
        { name: 'MAILLOT BMX ENFANT CONFORT ML', category: 'ENFANTS', type: 'enfant', subtype: 'Cyclisme Enfant', hasOptions: false, pricingTiers: [{ quantity: 1, price: 50.40 }] },
        { name: 'MAILLOT MI-SAISON ENFANT CONFORT ML', category: 'ENFANTS', type: 'enfant', subtype: 'Cyclisme Enfant', hasOptions: false, pricingTiers: [{ quantity: 1, price: 49.20 }] },
        { name: 'GILET COUPE-VENT LEGER ENFANT', category: 'ENFANTS', type: 'enfant', subtype: 'Cyclisme Enfant', hasOptions: false, pricingTiers: [{ quantity: 1, price: 42.00 }] },
        { name: 'VESTE HIVER ENFANT CONFORT', category: 'ENFANTS', type: 'enfant', subtype: 'Cyclisme Enfant', hasOptions: false, pricingTiers: [{ quantity: 1, price: 90.00 }] },
        { name: 'CUISSARD ENFANT CONFORT', category: 'ENFANTS', type: 'enfant', subtype: 'Cyclisme Enfant', hasOptions: false, pricingTiers: [{ quantity: 1, price: 42.00 }] },
        { name: 'COLLANT HIVER ENFANT SUBLIME CONFORT', category: 'ENFANTS', type: 'enfant', subtype: 'Cyclisme Enfant', hasOptions: false, pricingTiers: [{ quantity: 1, price: 60.00 }] },
        { name: 'COLLANT ENFANT VELOURS ECHAUFFEMENT', category: 'ENFANTS', type: 'enfant', subtype: 'Cyclisme Enfant', hasOptions: false, pricingTiers: [{ quantity: 1, price: 48.00 }] },
        { name: 'MAILLOT RUNNING ENFANT MANCHES COURTES', category: 'ENFANTS', type: 'enfant', subtype: 'Running Enfants', hasOptions: false, pricingTiers: [{ quantity: 1, price: 30.00 }] },
        { name: 'DEBARDEUR ATHLETISME ENFANT', category: 'ENFANTS', type: 'enfant', subtype: 'Running Enfants', hasOptions: false, pricingTiers: [{ quantity: 1, price: 27.00 }] },
        { name: 'CUISSARD RUNNING ENFANT', category: 'ENFANTS', type: 'enfant', subtype: 'Running Enfants', hasOptions: false, pricingTiers: [{ quantity: 1, price: 36.00 }] },
        { name: 'TRIFONCTION ENFANT COURTE DISTANCE', category: 'ENFANTS', type: 'enfant', subtype: 'Running Enfants', hasOptions: false, pricingTiers: [{ quantity: 1, price: 55.20 }] },
        { name: 'POCHE DOS ZIPPEE', category: 'option', type: 'option', pricingTiers: [ { quantity: 1, price: 11.72 }, { quantity: 2, price: 9.38 }, { quantity: 3, price: 7.50 }, { quantity: 5, price: 6.00 }, { quantity: 15, price: 5.70 }, { quantity: 25, price: 5.52 }, { quantity: 50, price: 5.40 }, { quantity: 80, price: 5.22 }, { quantity: 150, price: 5.10 } ] },
        { name: 'BANDE REFLECTIVE', category: 'option', type: 'option', pricingTiers: [ { quantity: 1, price: 7.08 }, { quantity: 2, price: 5.40 }, { quantity: 3, price: 4.50 }, { quantity: 5, price: 3.60 }, { quantity: 15, price: 3.42 }, { quantity: 25, price: 3.31 }, { quantity: 50, price: 3.24 }, { quantity: 80, price: 3.13 }, { quantity: 150, price: 3.06 }, ] },
        { name: 'Ajustement Longueur +3cm', category: 'option', type: 'option', optionGroup: 'length', fixedPriceTTC: 7.20 },
        { name: 'Ajustement Longueur +6cm', category: 'option', type: 'option', optionGroup: 'length', fixedPriceTTC: 7.20 },
        { name: 'Ajustement Longueur +9cm', category: 'option', type: 'option', optionGroup: 'length', fixedPriceTTC: 7.20 },
        { name: 'Ajustement Longueur -3cm', category: 'option', type: 'option', optionGroup: 'length', fixedPriceTTC: 7.20 },
        { name: 'Ajustement Longueur -6cm', category: 'option', type: 'option', optionGroup: 'length', fixedPriceTTC: 7.20 },
        { name: 'Ajustement Longueur -9cm', category: 'option', type: 'option', optionGroup: 'length', fixedPriceTTC: 7.20 },
    ];

    const productSizes = {
        haut: ['XXS', 'XS', 'S', 'M', 'L', 'XL', 'XXL', '3XL', '4XL', '5XL', '6XL'],
        enfant: ['6A', '8A', '10A', '12A', '14A', '16A'],
        aero: ['XXS', 'XS', 'S', 'M', 'L', 'XL', 'XXL', '3XL'],
        ample: ['XXS', 'XS', 'S', 'M', 'L', 'XL', 'XXL', '3XL', '4XL', '5XL', '6XL'],
        largeHaut: ['XXS', 'XS', 'S', 'M', 'L', 'XL', 'XXL', '3XL', '4XL', '5XL', '6XL'],
        manchettes: ["P (Biceps 27/31cm)", "G (Biceps 31/34cm)"],
        jambieres: ["P (Cuisses 39/44cm)", "G (Cuisses 44/50cm)"],
        unique: ["U"],
        gants: ["XXS", "XS", "S", "M", "L", "XL", "XXL"],
        chaussettes: ["S/M (35/40)", "L/XL (41/46)"],
        sousMaillot: ["2XS/XS", "S/M", "L/XL", "2XL/3XL"],
        sousMaillotHiver: ["S", "M", "L", "XL"],
        gantsMiSaison: ["S", "M", "L", "XL"],
        couvreChaussuresAero: ["36/38", "39/41", "42/44", "45/47"],
        couvreChaussuresHiver: ["38/39", "40/42", "43/44", "45/46", "47/48"],
        bandeau: ["XXS", "XS", "S", "M", "L", "XL", "XXL"],
        tourDeCou: ["XXS", "XS", "S", "M", "L", "XL", "XXL"],
    };

    const TVA_RATE = 0.20;
    const DOWN_PAYMENT_RATE = 0.30;
    const GRAPHIC_FEE_TTC = 150;
    
    const productMap = new Map(allAvailableProducts.map(p => [p.name, p]));

    // =================================================================================
    // --- APPLICATION STATE ---
    // =================================================================================
    let state = {
        currentStep: 1,
        isCustomCreation: false,
        applyDiscount: false,
        clubName: '',
        departement: '',
        clientNumber: '',
        orderDate: new Date().toISOString().split('T')[0],
        clubDiscount: 0,
        currentOrderLineItems: [],
        orderScope: 'global', // Simplifié: toujours global
    };
    
    // =================================================================================
    // --- DOM ELEMENT REFERENCES ---
    // =================================================================================
    const dom = {
        // Wizard elements
        wizardProgressBar: document.getElementById('wizard-progress-bar'),
        step1: document.getElementById('step-1'),
        step2: document.getElementById('step-2'),
        step3: document.getElementById('step-3'),

        // Step 1 elements
        clubName: document.getElementById('clubName'),
        departement: document.getElementById('departement'),
        clientNumber: document.getElementById('clientNumber'),
        orderDate: document.getElementById('orderDate'),
        
        // Step 2 elements
        productTabsContainer: document.getElementById('product-tabs-container'),
        productGrid: document.getElementById('product-grid'),
        orderSummaryContainer: document.getElementById('order-summary-container'),
        summaryGrandTotalTTC: document.getElementById('summary-grand-total-ttc'),
        emptyCartMessage: document.getElementById('empty-cart-message'),
        
        // Step 3 elements
        orderTableHead: document.getElementById('order-table-head'),
        orderTableBody: document.getElementById('order-table-body'),
        applyDiscountCheck: document.getElementById('apply-discount-check'),
        customCreationCheck: document.getElementById('custom-creation-check'),
        discountControlsContainer: document.getElementById('discount-controls-container'),
        clubDiscount: document.getElementById('clubDiscount'),
        subtotalHT: document.getElementById('subtotal-ht'),
        subtotalTTC: document.getElementById('subtotal-ttc'),
        discountAmountTTC: document.getElementById('discount-amount-ttc'),
        shippingTTC: document.getElementById('shipping-ttc'),
        graphicFeeContainer: document.getElementById('graphic-fee-container'),
        graphicFeeTTC: document.getElementById('graphic-fee-ttc'),
        grandTotalTTC: document.getElementById('grand-total-ttc'),
        downPaymentContainer: document.getElementById('down-payment-container'),
        validateOrderBtn: document.getElementById('validate-order-btn'),

        // Modals & Toasts
        mainModal: document.getElementById('main-modal'),
        mainModalTitle: document.getElementById('main-modal-title'),
        mainModalBody: document.getElementById('main-modal-body'),
        mainModalActions: document.getElementById('main-modal-actions'),
        toastContainer: document.getElementById('toast-container'),
    };
    
    // =================================================================================
    // --- HELPER & UTILITY FUNCTIONS ---
    // =================================================================================
    const showToast = (message, type = 'success') => {
        const toast = document.createElement('div');
        const bgColor = type === 'success' ? 'bg-green-500' : type === 'error' ? 'bg-red-500' : 'bg-blue-500';
        toast.className = `toast ${bgColor} text-white p-4 rounded-lg shadow-lg mb-2`;
        toast.textContent = message;
        dom.toastContainer.appendChild(toast);
        setTimeout(() => toast.classList.add('show'), 10);
        setTimeout(() => {
            toast.classList.remove('show');
            toast.addEventListener('transitionend', () => toast.remove());
        }, 3000);
    };

    const showModal = (title, content, actions = [], maxWidth = 'max-w-md') => {
        const modal = dom.mainModal;
        const modalDialog = modal.querySelector('.modal-content');
        modalDialog.className = `modal-content bg-white rounded-lg shadow-xl p-6 w-full relative ${maxWidth}`;
        dom.mainModalTitle.textContent = title;
        dom.mainModalBody.innerHTML = '';
        if (typeof content === 'string') {
            dom.mainModalBody.innerHTML = content;
        } else {
            dom.mainModalBody.appendChild(content);
        }
        dom.mainModalActions.innerHTML = '';
        actions.forEach(action => {
            const button = document.createElement('button');
            button.textContent = action.label;
            button.className = `${action.className || 'bg-indigo-600 hover:bg-indigo-700 text-white'} font-bold py-2 px-4 rounded-lg`;
            button.onclick = action.onClick;
            dom.mainModalActions.appendChild(button);
        });
        modal.classList.add('is-open');
    };

    const hideModal = () => dom.mainModal.classList.remove('is-open');
    
    // =================================================================================
    // --- WIZARD NAVIGATION ---
    // =================================================================================
    const navigateToStep = (stepNumber) => {
        state.currentStep = stepNumber;
        document.querySelectorAll('.wizard-step').forEach(step => step.classList.remove('active'));
        document.getElementById(`step-${stepNumber}`).classList.add('active');

        const progressPercentage = { 1: 33, 2: 66, 3: 100 }[stepNumber] || 33;
        dom.wizardProgressBar.style.width = `${progressPercentage}%`;
        
        if (stepNumber === 2) {
            renderProductGrid();
        }
        if (stepNumber === 3) {
            renderOrderTable();
            renderAllTotals();
        }
        window.scrollTo(0, 0);
    };

    // =================================================================================
    // --- CALCULATION LOGIC ---
    // =================================================================================
    const getPriceForQuantity = (pricingTiers, totalQuantity) => {
        if (!pricingTiers || pricingTiers.length === 0) return 0;
        let applicableTier = pricingTiers[0];
        for (let i = pricingTiers.length - 1; i >= 0; i--) {
            if (totalQuantity >= pricingTiers[i].quantity) {
                applicableTier = pricingTiers[i];
                break;
            }
        }
        return applicableTier.price;
    };

    const getUnitPriceTTC = (productName, totalPricingQuantity, selectedOptions = []) => {
        const product = productMap.get(productName);
        if (!product) return 0;
        const basePrice = product.price ? product.price : getPriceForQuantity(product.pricingTiers, totalPricingQuantity);
        const optionsPrice = selectedOptions.reduce((total, optionName) => {
            const optionProduct = productMap.get(optionName);
            if (!optionProduct) return total;
            if (optionProduct.fixedPriceTTC) return total + optionProduct.fixedPriceTTC;
            return total;
        }, 0);
        return basePrice + optionsPrice;
    };

    const calculateAllTotals = () => {
        const updatedLineItems = state.currentOrderLineItems.map(item => {
            const product = productMap.get(item.productName);
            if (!product) return { ...item, unitPriceTTC: 0, unitPriceHT: 0, totalPriceTTC: 0, totalPriceHT: 0 };

            let pricingQuantity = item.totalQuantity;
            if (product.pricingGroup) {
                pricingQuantity = state.currentOrderLineItems
                    .filter(li => productMap.get(li.productName)?.pricingGroup === product.pricingGroup)
                    .reduce((sum, li) => sum + li.totalQuantity, 0);
            }
            
            const finalUnitPriceTTC = getUnitPriceTTC(item.productName, pricingQuantity, item.options);
            const totalPriceTTC = finalUnitPriceTTC * item.totalQuantity;
            const finalUnitPriceHT = finalUnitPriceTTC / (1 + TVA_RATE);
            const totalPriceHT = totalPriceTTC / (1 + TVA_RATE);

            return { ...item, unitPriceTTC: finalUnitPriceTTC, unitPriceHT: finalUnitPriceHT, totalPriceTTC, totalPriceHT };
        });

        state.currentOrderLineItems = updatedLineItems;

        const originalSubtotalHT = updatedLineItems.reduce((acc, item) => acc + item.totalPriceHT, 0);
        const originalSubtotalTTC = updatedLineItems.reduce((acc, item) => acc + item.totalPriceTTC, 0);

        const discountAmountHT = state.applyDiscount ? originalSubtotalHT * (state.clubDiscount / 100) : 0;
        const discountAmountTTC = discountAmountHT * (1 + TVA_RATE);

        let shippingHT = 0;
        if (originalSubtotalHT > 0 && originalSubtotalHT <= 500) shippingHT = 9.50;
        else if (originalSubtotalHT <= 1000) shippingHT = 12;
        else if (originalSubtotalHT <= 2000) shippingHT = 14;
        
        const graphicFeeTTC = state.isCustomCreation ? GRAPHIC_FEE_TTC : 0;
        const graphicFeeHT = graphicFeeTTC / (1 + TVA_RATE);

        const shippingTTC = shippingHT * (1 + TVA_RATE);
        const grandTotalTTC = originalSubtotalTTC + shippingTTC + graphicFeeTTC - discountAmountTTC;
        
        return { subtotalHT: originalSubtotalHT, subtotalTTC: originalSubtotalTTC, discountAmountTTC, shippingTTC, graphicFeeTTC, grandTotalTTC };
    };

    // =================================================================================
    // --- UI RENDER FUNCTIONS ---
    // =================================================================================
    const renderAllTotals = () => {
        const totals = calculateAllTotals();
        dom.subtotalHT.textContent = `${totals.subtotalHT.toFixed(2)}€`;
        dom.subtotalTTC.textContent = `${totals.subtotalTTC.toFixed(2)}€`;
        dom.discountAmountTTC.textContent = `-${totals.discountAmountTTC.toFixed(2)}€`;
        dom.shippingTTC.textContent = `${totals.shippingTTC.toFixed(2)}€`;
        dom.graphicFeeContainer.classList.toggle('hidden', !state.isCustomCreation);
        dom.graphicFeeTTC.textContent = `${totals.graphicFeeTTC.toFixed(2)}€`;
        dom.grandTotalTTC.textContent = `${totals.grandTotalTTC.toFixed(2)}€`;
        dom.summaryGrandTotalTTC.textContent = `${totals.grandTotalTTC.toFixed(2)}€`;
    };

    const renderProductGrid = () => {
        const activeTab = document.querySelector('.product-tab-btn.border-indigo-500').dataset.tab;
        const productsToShow = allAvailableProducts.filter(p => {
            if (p.category === 'option') return false;
            const tabCategory = activeTab.replace('GAMME ', '');
            if (tabCategory === 'CYCLISME/RUNNING') {
                return p.category === 'CYCLISME' || p.category === 'RUNNING';
            }
            return p.category.toUpperCase() === tabCategory.toUpperCase();
        });

        dom.productGrid.innerHTML = productsToShow.map(p => `
            <div class="product-card bg-white border rounded-lg p-4 flex flex-col justify-between cursor-pointer" data-product-name="${p.name}">
                <div>
                    <h4 class="font-bold text-gray-800">${p.name}</h4>
                    <p class="text-xs text-gray-500">${p.subtype}</p>
                </div>
                <button class="add-product-btn mt-4 w-full px-3 py-2 bg-indigo-100 text-indigo-700 text-sm font-semibold rounded-md hover:bg-indigo-200">Ajouter</button>
            </div>
        `).join('');
    };

    const renderOrderSummary = () => {
        if (state.currentOrderLineItems.length === 0) {
            dom.emptyCartMessage.style.display = 'block';
            dom.orderSummaryContainer.innerHTML = '';
            dom.orderSummaryContainer.appendChild(dom.emptyCartMessage);
        } else {
            dom.emptyCartMessage.style.display = 'none';
            dom.orderSummaryContainer.innerHTML = state.currentOrderLineItems.map(item => `
                <div class="bg-slate-50 p-3 rounded-md">
                    <div class="flex justify-between items-start">
                        <div>
                            <p class="font-semibold text-sm">${item.productName}</p>
                            <p class="text-xs text-gray-500">Quantité : ${item.totalQuantity}</p>
                        </div>
                        <p class="font-bold text-sm whitespace-nowrap">${item.totalPriceTTC.toFixed(2)}€</p>
                    </div>
                     <div class="text-right mt-1">
                        <button class="remove-item-btn text-xs text-red-500 hover:text-red-700" data-item-id="${item.id}">Supprimer</button>
                    </div>
                </div>
            `).join('');
        }
    };

    const renderOrderTable = () => {
        dom.orderTableHead.innerHTML = `<tr>
            <th class="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase">Produit</th>
            <th class="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase">Qté</th>
            <th class="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase">Prix U. TTC</th>
            <th class="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase">Total TTC</th>
        </tr>`;
        dom.orderTableBody.innerHTML = state.currentOrderLineItems.map(item => `
            <tr>
                <td class="px-6 py-4"><div class="font-medium">${item.productName}</div></td>
                <td class="px-6 py-4">${item.totalQuantity}</td>
                <td class="px-6 py-4">${item.unitPriceTTC.toFixed(2)}€</td>
                <td class="px-6 py-4 font-bold">${item.totalPriceTTC.toFixed(2)}€</td>
            </tr>
        `).join('');
    };
    
    // =================================================================================
    // --- EVENT HANDLERS ---
    // =================================================================================
    
    // Navigation
    document.getElementById('next-to-step-2').addEventListener('click', () => {
        if (!state.clubName || !state.departement) {
            showToast('Veuillez renseigner le nom du club et le département.', 'error');
            return;
        }
        navigateToStep(2);
    });
    document.getElementById('back-to-step-1').addEventListener('click', () => navigateToStep(1));
    document.getElementById('next-to-step-3').addEventListener('click', () => navigateToStep(3));
    document.getElementById('back-to-step-2').addEventListener('click', () => navigateToStep(2));
    
    // Step 1 Inputs
    dom.clubName.addEventListener('change', e => state.clubName = e.target.value);
    dom.departement.addEventListener('change', e => state.departement = e.target.value);
    dom.clientNumber.addEventListener('change', e => state.clientNumber = e.target.value);
    dom.orderDate.addEventListener('change', e => state.orderDate = e.target.value);
    
    // Step 2 Interactions
    dom.productTabsContainer.addEventListener('click', e => {
        if (e.target.classList.contains('product-tab-btn')) {
            document.querySelectorAll('.product-tab-btn').forEach(btn => btn.classList.remove('border-indigo-500', 'text-indigo-600'));
            e.target.classList.add('border-indigo-500', 'text-indigo-600');
            renderProductGrid();
        }
    });

    dom.productGrid.addEventListener('click', e => {
        const card = e.target.closest('.product-card, .add-product-btn');
        if (!card) return;

        const productName = card.closest('.product-card').dataset.productName;
        const product = productMap.get(productName);
        if (!product) return;
        
        const content = document.createElement('div');
        const quantityInput = `
            <div class="flex items-center justify-between">
                <label for="modal-quantity" class="text-sm font-medium text-gray-700">Quantité totale</label>
                <input type="number" id="modal-quantity" class="modal-quantity-input mt-1 block w-24 rounded-md border-gray-300 shadow-sm text-center" placeholder="0">
            </div>
        `;

        content.innerHTML = `<div class="space-y-3">${quantityInput}</div>`;
        
        showModal(`Quantité pour ${product.name}`, content, [
            { label: 'Annuler', onClick: hideModal, className: 'bg-gray-200 text-gray-800' },
            { label: 'Ajouter au devis', onClick: () => {
                const totalQuantity = parseInt(content.querySelector('#modal-quantity').value, 10) || 0;

                if (totalQuantity > 0) {
                    state.currentOrderLineItems.push({
                        id: Date.now(),
                        productName: product.name,
                        quantitiesPerSize: { 'Qté': totalQuantity }, // Gardé pour la structure, mais ne contient plus de tailles
                        totalQuantity,
                        licencieName: '',
                        options: [],
                    });

                    renderAllTotals();
                    renderOrderSummary();
                    showToast(`${product.name} ajouté au devis.`);
                }
                hideModal();
            }, className: 'bg-indigo-600' }
        ]);
    });

    dom.orderSummaryContainer.addEventListener('click', e => {
        if (e.target.classList.contains('remove-item-btn')) {
            const itemId = e.target.dataset.itemId;
            state.currentOrderLineItems = state.currentOrderLineItems.filter(item => item.id != itemId);
            renderAllTotals();
            renderOrderSummary();
        }
    });

    // Step 3 Interactions
    dom.applyDiscountCheck.addEventListener('change', e => {
        state.applyDiscount = e.target.checked;
        dom.discountControlsContainer.classList.toggle('hidden', !state.applyDiscount);
        renderAllTotals();
    });
    dom.customCreationCheck.addEventListener('change', e => {
        state.isCustomCreation = e.target.checked;
        renderAllTotals();
    });
    dom.clubDiscount.addEventListener('change', e => {
        state.clubDiscount = parseFloat(e.target.value) || 0;
        renderAllTotals();
    });
    dom.validateOrderBtn.addEventListener('click', () => {
        if (state.currentOrderLineItems.length === 0) {
            showToast("Votre devis est vide.", "error");
            return;
        }
        showModal(
            "Validation finale",
            "<p>Vous êtes sur le point de générer le devis. Un PDF sera créé.</p>",
            [
                {label: "Annuler", onClick: hideModal, className: "bg-gray-300 text-black"},
                {label: "Générer le PDF", onClick: () => {
                    handleExportPdf();
                    hideModal();
                }, className: "bg-green-600"}
            ]
        )
    });


    // PDF & Excel Export Logic
    const handleExportPdf = () => {
        const { jsPDF } = window.jspdf;
        const doc = new jsPDF();
        doc.text(`Devis pour ${state.clubName}`, 10, 10);
        
        const head = [['Produit', 'Qté', 'Prix U. TTC', 'Total TTC']];
        const body = state.currentOrderLineItems.map(item => [
            item.productName,
            item.totalQuantity,
            `${item.unitPriceTTC.toFixed(2)} €`,
            `${item.totalPriceTTC.toFixed(2)} €`,
        ]);

        doc.autoTable({ head, body, startY: 20 });
        
        const finalY = doc.autoTable.previous.finalY + 10;
        const totals = calculateAllTotals();
        
        doc.setFontSize(10);
        doc.text(`Sous-total TTC: ${totals.subtotalTTC.toFixed(2)} €`, 150, finalY, { align: 'right' });
        if(state.applyDiscount) {
             doc.text(`Remise: -${totals.discountAmountTTC.toFixed(2)} €`, 150, finalY + 5, { align: 'right' });
        }
        if(state.isCustomCreation) {
            doc.text(`Forfait Création: ${totals.graphicFeeTTC.toFixed(2)} €`, 150, finalY + 10, { align: 'right' });
        }
        doc.text(`Frais de port TTC: ${totals.shippingTTC.toFixed(2)} €`, 150, finalY + 15, { align: 'right' });
        doc.setFontSize(12);
        doc.setFont(undefined, 'bold');
        doc.text(`Total TTC: ${totals.grandTotalTTC.toFixed(2)} €`, 150, finalY + 25, { align: 'right' });


        doc.save(`devis_${state.clubName.replace(/ /g, '_')}.pdf`);
        showToast("PDF du devis exporté avec succès!", "success");
    };

    const handleExportExcel = () => {
        const ws = XLSX.utils.json_to_sheet(state.currentOrderLineItems.map(item => ({
            Produit: item.productName,
            Quantité: item.totalQuantity,
            'Prix Unitaire TTC': item.unitPriceTTC,
            'Total TTC': item.totalPriceTTC
        })));
        const wb = XLSX.utils.book_new();
        XLSX.utils.book_append_sheet(wb, ws, "Devis");
        XLSX.writeFile(wb, `devis_sauvegarde_${state.clubName.replace(/ /g, '_')}.xlsx`);
        showToast("Fichier de sauvegarde exporté avec succès!", "success");
    };

    document.getElementById('save-order-btn').addEventListener('click', handleExportExcel);
    
    // --- INITIALIZATION ---
    document.addEventListener('DOMContentLoaded', () => {
        const scopeContainer = document.querySelector('label[for="scope-global"]')?.parentElement?.parentElement;
        if(scopeContainer) scopeContainer.parentElement.remove();

        document.getElementById('main-modal-close-btn').addEventListener('click', hideModal);
        navigateToStep(1);
    });

    </script>
</body>
</html>

