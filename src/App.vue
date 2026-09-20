<script>
import themeToggleBtn from './components/ThemeToggle.vue'
import { initFlowbite } from 'flowbite'

export default {
  components: {
    themeToggleBtn
  },
  data() {
    return {
      // ⚠️ APNA GOOGLE APPS SCRIPT WEB APP URL YAHAN DALEIN
      gasWebhookUrl: 'YOUR_GOOGLE_APPS_SCRIPT_WEB_APP_URL',
      currency: 'INR',

      isOnline: navigator.onLine,
      isSyncing: false,

      // Dropdown Options
      dropdowns: {
        descriptions: ['D1', 'D2', 'D3', 'D4', 'D5'],
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
        // TODO
        this.currency = 'INR' || 'OMR';
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
    initFlowbite();
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
  },
};
</script>


<template>
  <div class="bg-neutral-primary border-b border-default">
    <nav class="bg-neutral-primary sticky w-full z-20 top-0 inset-0 border-b border-default mb-2">
      <div class="max-w-full flex flex-wrap items-center justify-between p-4 mx-2">
        <a href="https://flowbite.com/" class="flex items-center space-x-3 rtl:space-x-reverse">
          <img src="https://flowbite.com/docs/images/logo.svg" class="h-7" alt="Flowbite Logo" />
          <span class="self-center text-xl text-heading font-semibold whitespace-nowrap">Flowbite</span>
        </a>
        <div class="inline-flex md:order-2 space-x-3 md:space-x-0 rtl:space-x-reverse">
          <span :class="isOnline ? 'bg-emerald-500' : 'bg-rose-500'"
            class="h-3 w-3 animate-pulse rounded-full place-self-center me-2"></span>

          <!-- <button type="button" v-if="unsyncedCount == 0" @click="syncOfflineData" :disabled="isSyncing || !isOnline"
          class="text-white bg-brand hover:bg-brand-strong box-border border border-transparent focus:ring-4 focus:ring-brand-medium shadow-xs font-medium leading-5 rounded-base text-sm px-3 py-2 focus:outline-none">
                  <span :class="isOnline ? 'bg-emerald-500' : 'bg-rose-500'"
          class="h-3 w-3 animate-pulse rounded-full place-self-center"></span>

          {{
            isSyncing ? 'Syncing...' : `Sync Now (${unsyncedCount})` }}</button> -->
          <themeToggleBtn />
          <button data-collapse-toggle="navbar-cta" type="button"
            class="inline-flex items-center p-2 w-9 h-9 justify-center text-sm text-body rounded-base md:hidden hover:bg-neutral-secondary-soft hover:text-heading focus:outline-none focus:ring-2 focus:ring-neutral-tertiary"
            aria-controls="navbar-cta" aria-expanded="false">
            <span class="sr-only">Open main menu</span>
            <svg class="w-6 h-6" aria-hidden="true" xmlns="http://www.w3.org/2000/svg" width="24" height="24"
              fill="none" viewBox="0 0 24 24">
              <path stroke="currentColor" stroke-linecap="round" stroke-width="2" d="M5 7h14M5 12h14M5 17h14" />
            </svg>
          </button>
        </div>
        <div class="items-center justify-between hidden w-full md:flex md:w-auto md:order-1" id="navbar-cta">
          <ul
            class="font-medium flex flex-col p-4 md:p-0 mt-4 border border-default rounded-base bg-neutral-secondary-soft md:flex-row md:space-x-8 rtl:space-x-reverse md:mt-0 md:border-0 md:bg-neutral-primary">
            <li>
              <a href="#" class="block py-2 px-3 text-white bg-brand rounded md:bg-transparent md:text-fg-brand md:p-0"
                aria-current="page">Home</a>
            </li>
            <li>
              <a href="#"
                class="block py-2 px-3 text-heading rounded hover:bg-neutral-tertiary md:hover:bg-transparent md:border-0 md:hover:text-fg-brand md:p-0 md:dark:hover:bg-transparent">About</a>
            </li>
            <li>
              <a href="#"
                class="block py-2 px-3 text-heading rounded hover:bg-neutral-tertiary md:hover:bg-transparent md:border-0 md:hover:text-fg-brand md:p-0 md:dark:hover:bg-transparent">Services</a>
            </li>
            <li>
              <a href="#"
                class="block py-2 px-3 text-heading rounded hover:bg-neutral-tertiary md:hover:bg-transparent md:border-0 md:hover:text-fg-brand md:p-0 md:dark:hover:bg-transparent">Contact</a>
            </li>
          </ul>
        </div>
      </div>
    </nav>

    <div class="grid grid-cols-1 gap-6 lg:grid-cols-5 mx-2">

      <section class="bg-neutral-primary-soft block p-6 border border-default rounded-base shadow-xs lg:col-span-2">
        <div class="flex justify-between">
          <h1 class="mb-5 text-xl font-bold tracking-tighter text-heading md:text-xl lg:text-2xl">Add New Entry</h1>
          <label class="inline-flex items-center mb-5 cursor-pointer">
            <input type="checkbox" true-value="INR" false-value="OMR" v-model="currency" class="sr-only peer">
            <div
              class="relative w-11 h-6 bg-red-500 peer-focus:outline-none peer-focus:ring-4 peer-focus:ring-brand-soft dark:peer-focus:ring-brand-soft rounded-full peer peer-checked:after:translate-x-full rtl:peer-checked:after:-translate-x-full peer-checked:after:border-buffer after:content-[''] after:absolute after:top-0.5 after:inset-0.5 after:bg-white after:rounded-full after:h-5 after:w-5 after:transition-all peer-checked:bg-orange-500">
            </div>
            <span class="select-none ms-3 text-sm font-medium text-heading">{{ currency }}</span>
          </label>
        </div>
        <form @submit.prevent="handleSubmit" class="grid gap-6 mb-6">
          <!-- Date -->
          <div>
            <label for="date" class="block mb-2.5 text-sm font-medium text-heading">Date</label>
            <input type="date" id="date" v-model="form.date" required
              class="bg-neutral-secondary-medium border border-default-medium text-heading text-sm rounded-base focus:ring-brand focus:border-brand block w-full px-3 py-2.5 shadow-xs placeholder:text-body" />
          </div>
          <div class="grid md:grid-cols-2 gap-6">
            <!-- Category (Dropdown) -->
            <div>
              <label class="block mb-2.5 text-sm font-medium text-heading">Category</label>
              <select v-model="form.category" required
                class="bg-neutral-secondary-medium border border-default-medium text-heading text-sm rounded-base focus:ring-brand focus:border-brand block w-full px-3 py-2.5 shadow-xs placeholder:text-body">
                <option value="" disabled>Select Category</option>
                <option v-for="cat in dropdowns.categories" :key="cat" :value="cat">{{ cat }}</option>
              </select>
            </div>

            <!-- Description (Dropdown) -->
            <div>
              <label class="block mb-2.5 text-sm font-medium text-heading">Description</label>
              <select v-model="form.description" required
                class="bg-neutral-secondary-medium border border-default-medium text-heading text-sm rounded-base focus:ring-brand focus:border-brand block w-full px-3 py-2.5 shadow-xs placeholder:text-body">
                <option value="" disabled>Select Description</option>
                <option v-for="desc in dropdowns.descriptions" :key="desc" :value="desc">{{ desc }}</option>
              </select>
            </div>

          </div>
          <!-- Platform (Dropdown) -->
          <div class="grid md:grid-cols-2 gap-6">
            <div>
              <label class="block mb-2.5 text-sm font-medium text-heading">Platform</label>
              <select v-model="form.platform" required
                class="bg-neutral-secondary-medium border border-default-medium text-heading text-sm rounded-base focus:ring-brand focus:border-brand block w-full px-3 py-2.5 shadow-xs placeholder:text-body">
                <option value="" disabled>Select Platform</option>
                <option v-for="plat in dropdowns.platforms" :key="plat" :value="plat">{{ plat }}</option>
              </select>
            </div>

            <!-- Person (Dropdown) -->
            <div>
              <label class="block mb-2.5 text-sm font-medium text-heading">Person</label>
              <select v-model="form.person" required
                class="bg-neutral-secondary-medium border border-default-medium text-heading text-sm rounded-base focus:ring-brand focus:border-brand block w-full px-3 py-2.5 shadow-xs placeholder:text-body">
                <option value="" disabled>Select Person</option>
                <option v-for="per in dropdowns.persons" :key="per" :value="per">{{ per }}</option>
              </select>
            </div>
          </div>

          <!-- Credit / Debit Grid -->
          <div class=" grid md:grid-cols-2 gap-6">
            <div>
              <label class="block mb-2.5 text-sm font-medium text-heading">Credit (+)</label>
              <input type="number" v-model.number="form.credit" min="0" placeholder="0"
                class="bg-neutral-secondary-medium border border-default-medium text-heading text-sm rounded-base focus:ring-brand focus:border-brand block w-full px-3 py-2.5 shadow-xs placeholder:text-body" />
            </div>
            <div>
              <label class="block mb-2.5 text-sm font-medium text-heading">Debit (-)</label>
              <input type="number" v-model.number="form.debit" min="0" placeholder="0"
                class="bg-neutral-secondary-medium border border-default-medium text-heading text-sm rounded-base focus:ring-brand focus:border-brand block w-full px-3 py-2.5 shadow-xs placeholder:text-body" />
            </div>
          </div>

          <!-- Submit Button -->
          <button type="submit"
            class="text-white bg-brand box-border border border-transparent hover:bg-brand-strong focus:ring-4 focus:ring-brand-medium shadow-xs font-medium leading-5 rounded-base text-sm px-4 py-2.5 focus:outline-none">
            Add Entry
          </button>
        </form>
      </section>

      <section
        class="bg-neutral-primary-soft block p-6 border border-default rounded-base shadow-xs    lg:col-span-3 overflow-hidden">
        <div class="mb-4 flex justify-between items-center">
          <h2 class="text-lg font-bold text-slate-900">Recent Transactions</h2>
          <div class="text-right">
            <p class="text-xs text-body font-medium">Total Running Balance</p>
            <p class="text-xl font-black text-teal">{{ currency }} {{ totalRunningBalance }}</p>
          </div>
        </div>

        <div class="relative overflow-x-auto bg-neutral-primary-soft shadow-xs rounded-base border border-default">
          <table class="w-full text-sm text-left rtl:text-right text-heading">
            <thead class="bg-neutral-secondary-soft border-b border-default">
              <tr>
                <th scope="col" class="px-6 py-3 font-bold">
                  Date
                </th>
                <th scope="col" class="px-6 py-3 font-bold">
                  Description
                </th>
                <th scope="col" class="px-6 py-3 font-bold">
                  Category
                </th>
                <th scope="col" class="px-6 py-3 font-bold">
                  Platform
                </th>
                <th scope="col" class="px-6 py-3 font-bold">
                  Person
                </th>
                <th scope="col" class="px-6 py-3 font-bold">
                  Credit
                </th>
                <th scope="col" class="px-6 py-3 font-bold">
                  Debit
                </th>
                <th scope="col" class="px-6 py-3 font-bold">
                  Balance
                </th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="(row, index) in calculatedTransactions" :key="index"
                class="odd:bg-neutral-primary even:bg-neutral-secondary-soft border-b border-default">
                <!-- <th scope="row" class="px-6 py-4 font-medium text-heading whitespace-nowrap">
                  Apple MacBook Pro 17"
                </th> -->
                <td class="px-6 py-4 whitespace-nowrap">{{ row.date }}</td>
                <td class="px-6 py-4">{{ row.description }}</td>
                <td class="px-6 py-4">{{ row.category }}</td>
                <td class="px-6 py-4">{{ row.platform }}</td>
                <td class="px-6 py-4">{{ row.person }}</td>
                <td class="px-6 py-4 text-emerald-600 font-semibold">+{{ row.credit }}</td>
                <td class="px-6 py-4 text-rose-600 font-semibold">-{{ row.debit }}</td>
                <td class="px-6 py-4 font-bold text-fg-brand"><span>{{ currency }}</span> {{ row.runningBalance }}</td>
              </tr>
              <tr v-if="transactions.length == 0">
                <td colspan="8"
                  class="p-8 text-body text-center bg-neutral-secondary-medium font-medium leading-5 text-sm px-4 py-2.5 focus:outline-none">
                  No entries yet. Start adding</td>
              </tr>
            </tbody>
          </table>
        </div>
      </section>
    </div>
  </div>
</template>