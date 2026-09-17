<template>
  <div class="min-h-screen bg-slate-50 p-4 font-sans text-slate-800">
    <!-- Header & Sync Status -->
    <header
      class="mb-6 flex flex-col justify-between gap-4 rounded-2xl bg-white p-6 shadow-sm sm:flex-row sm:items-center">
      <div>
        <h1 class="text-2xl font-bold text-slate-900">exbuddy</h1>
        <!-- <p class="text-sm text-slate-500">Google Sheets Sync WebApp</p> -->
      </div>

      <!-- Network Indicator -->
      <div class="flex items-center gap-3">
        <span :class="isOnline ? 'bg-emerald-500' : 'bg-rose-500'" class="h-3 w-3 animate-pulse rounded-full"></span>
        <span class="text-sm font-medium">
          {{ isOnline ? 'Online (Auto-Sync On)' : 'Offline (Saving Locally)' }}
        </span>
        <button v-if="unsyncedCount > 0" @click="syncOfflineData" :disabled="isSyncing || !isOnline"
          class="cursor-pointer rounded-lg bg-indigo-600 px-4 py-2 text-xs font-semibold text-white shadow-sm hover:bg-indigo-500 disabled:bg-slate-300">
          {{ isSyncing ? 'Syncing...' : `Sync Now (${unsyncedCount})` }}
        </button>
      </div>
    </header>

    <div class="grid grid-cols-1 gap-6 lg:grid-cols-3">
      <!-- 1. ENTRY FORM (WhatsApp Modern Style Simple Input) -->
      <section class="rounded-2xl bg-white p-6 shadow-sm lg:col-span-1">
        <h2 class="mb-4 text-lg font-bold text-slate-900">New Entry</h2>

        <form @submit.prevent="handleSubmit" class="space-y-4">
          <!-- Date -->
          <div>
            <label class="mb-1 block text-xs font-semibold text-slate-600">Date</label>
            <input type="date" v-model="form.date" required
              class="w-full rounded-xl border border-slate-200 p-3 text-sm focus:border-indigo-500 focus:outline-none" />
          </div>

          <!-- Description -->
          <div>
            <label class="mb-1 block text-xs font-semibold text-slate-600">Description</label>
            <input type="text" v-model="form.description" placeholder="e.g. Chai, Groceries" required
              class="w-full rounded-xl border border-slate-200 p-3 text-sm focus:border-indigo-500 focus:outline-none" />
          </div>

          <!-- Category (Dropdown) -->
          <div>
            <label class="mb-1 block text-xs font-semibold text-slate-600">Category</label>
            <select v-model="form.category" required
              class="w-full rounded-xl border border-slate-200 p-3 text-sm focus:border-indigo-500 focus:outline-none bg-white">
              <option value="" disabled>Select Category</option>
              <option v-for="cat in dropdowns.categories" :key="cat" :value="cat">{{ cat }}</option>
            </select>
          </div>

          <!-- Platform (Dropdown) -->
          <div>
            <label class="mb-1 block text-xs font-semibold text-slate-600">Platform</label>
            <select v-model="form.platform" required
              class="w-full rounded-xl border border-slate-200 p-3 text-sm focus:border-indigo-500 focus:outline-none bg-white">
              <option value="" disabled>Select Platform</option>
              <option v-for="plat in dropdowns.platforms" :key="plat" :value="plat">{{ plat }}</option>
            </select>
          </div>

          <!-- Person (Dropdown) -->
          <div>
            <label class="mb-1 block text-xs font-semibold text-slate-600">Person</label>
            <select v-model="form.person" required
              class="w-full rounded-xl border border-slate-200 p-3 text-sm focus:border-indigo-500 focus:outline-none bg-white">
              <option value="" disabled>Select Person</option>
              <option v-for="per in dropdowns.persons" :key="per" :value="per">{{ per }}</option>
            </select>
          </div>

          <!-- Credit / Debit Grid -->
          <div class="grid grid-cols-2 gap-4">
            <div>
              <label class="mb-1 block text-xs font-semibold text-slate-600">Credit (+)</label>
              <input type="number" v-model.number="form.credit" min="0" placeholder="0"
                class="w-full rounded-xl border border-slate-200 p-3 text-sm focus:border-indigo-500 focus:outline-none" />
            </div>
            <div>
              <label class="mb-1 block text-xs font-semibold text-slate-600">Debit (-)</label>
              <input type="number" v-model.number="form.debit" min="0" placeholder="0"
                class="w-full rounded-xl border border-slate-200 p-3 text-sm focus:border-indigo-500 focus:outline-none" />
            </div>
          </div>

          <!-- Submit Button -->
          <button type="submit"
            class="w-full cursor-pointer rounded-xl bg-indigo-600 p-3 text-sm font-bold text-white shadow-md hover:bg-indigo-500 transition-colors">
            Add Entry
          </button>
        </form>
      </section>

      <!-- 2. DATA TABLE & RUNNING BALANCE -->
      <section class="rounded-2xl bg-white p-6 shadow-sm lg:col-span-2 overflow-hidden">
        <div class="mb-4 flex justify-between items-center">
          <h2 class="text-lg font-bold text-slate-900">Recent Transactions</h2>
          <div class="text-right">
            <p class="text-xs text-slate-500 font-semibold">Total Running Balance</p>
            <p class="text-xl font-black text-indigo-600">₹ {{ totalRunningBalance }}</p>
          </div>
        </div>

        <!-- Responsive Table Wrapper -->
        <div class="overflow-x-auto">
          <table class="w-full text-left border-collapse text-sm">
            <thead>
              <tr class="border-b border-slate-100 bg-slate-50 text-xs font-bold uppercase text-slate-500">
                <th class="p-3">Date</th>
                <th class="p-3">Description</th>
                <th class="p-3">Category</th>
                <th class="p-3">Platform</th>
                <th class="p-3">Person</th>
                <th class="p-3 text-emerald-600">Credit</th>
                <th class="p-3 text-rose-600">Debit</th>
                <th class="p-3 bg-indigo-50 text-indigo-700">Balance</th>
              </tr>
            </thead>
            <tbody class="divide-y divide-slate-100">
              <tr v-for="(row, index) in calculatedTransactions" :key="index"
                class="hover:bg-slate-50/80 transition-colors">
                <td class="p-3 whitespace-nowrap">{{ row.date }}</td>
                <td class="p-3 font-medium text-slate-900">{{ row.description }}</td>
                <td class="p-3"><span class="rounded-full bg-slate-100 px-2.5 py-0.5 text-xs">{{ row.category }}</span>
                </td>
                <td class="p-3 text-slate-500">{{ row.platform }}</td>
                <td class="p-3 text-slate-500">{{ row.person }}</td>
                <td class="p-3 text-emerald-600 font-semibold">+{{ row.credit }}</td>
                <td class="p-3 text-rose-600 font-semibold">-{{ row.debit }}</td>
                <td class="p-3 bg-indigo-50/50 font-bold text-indigo-600">₹ {{ row.runningBalance }}</td>
              </tr>
              <tr v-if="transactions.length === 0">
                <td colspan="8" class="p-8 text-center text-slate-400">No entries yet. Start adding above!</td>
              </tr>
            </tbody>
          </table>
        </div>
      </section>
    </div>
  </div>
</template>

<script>
export default {
  data() {
    return {
      // ⚠️ APNA GOOGLE APPS SCRIPT WEB APP URL YAHAN DALEIN
      gasWebhookUrl: 'YOUR_GOOGLE_APPS_SCRIPT_WEB_APP_URL',

      isOnline: navigator.onLine,
      isSyncing: false,

      // Dropdown Options
      dropdowns: {
        categories: ['Food', 'Rent', 'Salary', 'Entertainment', 'Travel', 'Medical'],
        platforms: ['Cash', 'UPI / GPay', 'Credit Card', 'Bank Transfer'],
        persons: ['Self', 'Family', 'Roommate', 'Office']
      },

      // Single Form State
      form: {
        date: new Date().toISOString().split('T')[0], // Today's date default
        description: '',
        category: '',
        platform: '',
        person: '',
        credit: 0,
        debit: 0
      },

      // Main Data Store
      transactions: []
    };
  },

  computed: {
    // 🧮 Running Balance Auto-Calculation Logic
    calculatedTransactions() {
      let currentBalance = 0;
      return this.transactions.map(item => {
        currentBalance = currentBalance + (item.credit || 0) - (item.debit || 0);
        return {
          ...item,
          runningBalance: currentBalance
        };
      });
    },
    totalRunningBalance() {
      const len = this.calculatedTransactions.length;
      return len > 0 ? this.calculatedTransactions[len - 1].runningBalance : 0;
    },
    unsyncedCount() {
      return this.transactions.filter(t => t.synced === false).length;
    }
  },

  methods: {
    // 1. Handle Form Submit
    async handleSubmit() {
      const newEntry = {
        id: crypto.randomUUID ? crypto.randomUUID() : Date.now().toString(), // Unique ID for duplicate check
        date: this.form.date,
        description: this.form.description,
        category: this.form.category,
        platform: this.form.platform,
        person: this.form.person,
        credit: Number(this.form.credit) || 0,
        debit: Number(this.form.debit) || 0,
        synced: false // Initial state offline/unsynced
      };

      // Pehle UI update ke liye local state me push karo
      this.transactions.push(newEntry);
      this.saveToLocalStorage(); // Mobile LocalStorage me save kiya

      // Reset Form (keep date same for quick entries)
      this.form.description = '';
      this.form.category = '';
      this.form.platform = '';
      this.form.person = '';
      this.form.credit = 0;
      this.form.debit = 0;

      // Agar internet online hai toh turant sync trigger karo
      if (this.isOnline) {
        this.syncOfflineData();
      }
    },

    // 2. Google Sheets API Sync Logic
    async syncOfflineData() {
      if (this.isSyncing || !this.isOnline) return;


      const unsyncedRows = this.transactions.filter(t => !t.synced);
      if (unsyncedRows.length === 0) return;
      this.isSyncing = true;
      try {
        // Standard Fetch API Request for GAS
        const response = await fetch(this.gasWebhookUrl, {
          method: 'POST',
          mode: 'no-cors', // GAS requirements ke liye no-cors zaroori hota hai jab client-side direct request ho
          headers: {
            'Content-Type': 'application/json'
          },
          body: JSON.stringify({ entries: unsyncedRows })
        });
        // 'no-cors' me status check nahi ho pata par execution hit ho jati hai.
        // Best approach: Local entries ko synced mark kar do.
        this.transactions.forEach(t => {
          if (!t.synced) t.synced = true;
        });
        this.saveToLocalStorage();
      } catch (error) {
        console.error("Sync fail ho gaya:", error);
      } finally {
        this.isSyncing = false;
      }
    },
    // Local Storage helpers for Mobile Persistence
    saveToLocalStorage() {
      localStorage.setItem('budget_transactions', JSON.stringify(this.transactions));
    },
    loadFromLocalStorage() {
      const data = localStorage.getItem('budget_transactions');
      if (data) {
        this.transactions = JSON.parse(data);
      }
    },
    // Monitor Network Connection Changes
    updateOnlineStatus() {
      this.isOnline = navigator.onLine;
      if (this.isOnline) {
        this.syncOfflineData(); // Internet aate hi automatic sync!
      }
    }
  },
  mounted() {
    this.loadFromLocalStorage();
    window.addEventListener('online', this.updateOnlineStatus);
    window.addEventListener('offline', this.updateOnlineStatus);
    // Initial check sync if online
    if (this.isOnline) {
      this.syncOfflineData();
    }
  },
  unmounted() {
    window.removeEventListener('online', this.updateOnlineStatus);
    window.removeEventListener('offline', this.updateOnlineStatus);
  }
};
</script>