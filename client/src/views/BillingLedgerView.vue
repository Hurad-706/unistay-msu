<script setup lang="ts">
import { computed, onMounted, ref, watch } from 'vue';
import AppLayout from '../components/layout/AppLayout.vue';
import { useBillingStore } from '../stores/billing';
import { useNetworkStore } from '../stores/network';
import { Invoice, Payment } from '../types';
import {
  Calculator,
  CheckCircle2,
  Clock,
  CreditCard,
  Download,
  Droplet,
  Printer,
  Zap,
} from 'lucide-vue-next';

const billingStore = useBillingStore();
const networkStore = useNetworkStore();

const electricBill = ref(3600);
const waterBill = ref(1200);
const occupants = ref(8);
const splitResult = ref<any>(null);
const selectedInvoiceId = ref('');
const paymentAmount = ref(0);
const paymentMethod = ref<'cash' | 'gcash' | 'bank_transfer'>('gcash');
const paymentFeedback = ref<string | null>(null);
const paymentError = ref<string | null>(null);

const selectedInvoice = computed(() =>
  billingStore.invoices.find((invoice: Invoice) => invoice.id === selectedInvoiceId.value),
);
const selectedBalance = computed(() =>
  selectedInvoice.value
    ? Math.max(0, Number(selectedInvoice.value.totalAmount) - Number(selectedInvoice.value.paidAmount))
    : 0,
);

onMounted(() => {
  billingStore.fetchInvoices();
  calculateSplit();
});

watch(selectedInvoiceId, () => {
  paymentAmount.value = selectedBalance.value;
});

async function calculateSplit() {
  splitResult.value = await billingStore.calculateUtilitySplit(
    electricBill.value,
    waterBill.value,
    occupants.value,
  );
}

async function handlePayment() {
  paymentFeedback.value = null;
  paymentError.value = null;

  if (!selectedInvoiceId.value || paymentAmount.value <= 0 || paymentAmount.value > selectedBalance.value) {
    paymentError.value = 'Choose an invoice and enter an amount no greater than its outstanding balance.';
    return;
  }

  try {
    const result = await billingStore.recordPayment(
      selectedInvoiceId.value,
      paymentAmount.value,
      paymentMethod.value,
    );
    paymentFeedback.value = result.online
      ? 'Payment recorded and confirmed on server.'
      : result.message || 'Payment queued offline and will synchronize when reconnected.';
    paymentAmount.value = selectedBalance.value;
  } catch (error) {
    paymentError.value = error instanceof Error ? error.message : 'Unable to record payment.';
  }
}

function formatMoney(value: number | string) {
  return Number(value).toLocaleString('en-PH', {
    minimumFractionDigits: 2,
    maximumFractionDigits: 2,
  });
}

function formatDate(value: string | Date) {
  const date = new Date(value);
  return Number.isNaN(date.getTime())
    ? String(value)
    : date.toLocaleDateString('en-PH', { year: 'numeric', month: 'short', day: 'numeric' });
}

function outstandingBalance(invoice: Invoice) {
  return Math.max(0, Number(invoice.totalAmount) - Number(invoice.paidAmount));
}

function statusClass(status: Invoice['status']) {
  if (status === 'paid') return 'bg-emerald-100 text-emerald-800';
  if (status === 'partially_paid') return 'bg-amber-100 text-amber-800';
  if (status === 'overdue') return 'bg-red-100 text-red-800';
  return 'bg-slate-100 text-slate-700';
}

function csvCell(value: unknown) {
  const text = String(value ?? '');
  const safeText = /^[\s]*[=+\-@]/.test(text) ? `'${text}` : text;
  return `"${safeText.replace(/"/g, '""')}"`;
}

function downloadLedger() {
  const headers = [
    'Invoice number',
    'Property',
    'Unit',
    'Rent',
    'Electricity',
    'Water',
    'Other fees',
    'Total charges',
    'Total paid',
    'Outstanding balance',
    'Invoice status',
    'Due date',
    'Payment date',
    'Payment amount',
    'Payment method',
    'Payment status',
    'Receipt',
  ];
  const rows = billingStore.invoices.flatMap((invoice: Invoice) => {
    const payments: Array<Payment | undefined> = invoice.payments?.length ? invoice.payments : [undefined];
    return payments.map((payment: Payment | undefined) => [
      invoice.invoiceNumber,
      invoice.property?.name,
      invoice.unit?.roomNumber,
      invoice.rentAmount,
      invoice.electricityAmount,
      invoice.waterAmount,
      invoice.otherFees,
      invoice.totalAmount,
      invoice.paidAmount,
      outstandingBalance(invoice),
      invoice.status.replace('_', ' '),
      formatDate(invoice.dueDate),
      payment ? formatDate(payment.paymentDate) : '',
      payment?.amount,
      payment?.paymentMethod?.replace('_', ' '),
      payment ? (payment.isSynced ? 'Confirmed' : 'Awaiting sync') : '',
      payment?.receiptUrl,
    ]);
  });
  const csv = [headers, ...rows].map((row) => row.map(csvCell).join(',')).join('\r\n');
  const url = URL.createObjectURL(new Blob([csv], { type: 'text/csv;charset=utf-8' }));
  const link = document.createElement('a');
  link.href = url;
  link.download = 'unistay-rent-payment-ledger.csv';
  link.click();
  setTimeout(() => URL.revokeObjectURL(url), 0);
}

function printLedger() {
  window.print();
}
</script>

<template>
  <AppLayout>
    <div class="space-y-8">
      <div class="print-hidden">
        <h1 class="text-2xl font-bold text-slate-900">Rent & Payment Records</h1>
        <p class="text-xs text-slate-600">
          Review monthly charges, due dates, payment history, and current balances.
        </p>
      </div>

      <div class="grid grid-cols-1 gap-6 lg:grid-cols-3 print-hidden">
        <section class="space-y-4 rounded-xl border border-slate-200 bg-white p-5 shadow-sm">
          <div class="flex items-center space-x-2 border-b pb-2 text-sm font-bold text-slate-800">
            <Calculator class="h-4 w-4 text-msu-maroon" />
            <span>Shared Utility Splitter</span>
          </div>
          <div class="space-y-3 text-xs">
            <label class="block font-medium text-slate-600">
              <span class="mb-1 flex items-center"><Zap class="mr-1 h-3.5 w-3.5 text-amber-500" />Total dorm electric bill (₱)</span>
              <input v-model.number="electricBill" type="number" min="0" @input="calculateSplit" class="w-full rounded border border-slate-300 px-3 py-1.5 focus:outline-none focus:ring-1 focus:ring-msu-maroon" />
            </label>
            <label class="block font-medium text-slate-600">
              <span class="mb-1 flex items-center"><Droplet class="mr-1 h-3.5 w-3.5 text-blue-500" />Total dorm water bill (₱)</span>
              <input v-model.number="waterBill" type="number" min="0" @input="calculateSplit" class="w-full rounded border border-slate-300 px-3 py-1.5 focus:outline-none focus:ring-1 focus:ring-msu-maroon" />
            </label>
            <label class="block font-medium text-slate-600">
              <span class="mb-1 block">Active room occupants</span>
              <input v-model.number="occupants" type="number" min="1" @input="calculateSplit" class="w-full rounded border border-slate-300 px-3 py-1.5 focus:outline-none focus:ring-1 focus:ring-msu-maroon" />
            </label>
            <div v-if="splitResult" class="rounded-lg border border-emerald-200 bg-emerald-50 p-3 text-emerald-900">
              <div class="text-xs font-bold uppercase text-emerald-800">Calculated share per student</div>
              <div class="mt-1 text-xl font-extrabold text-emerald-700">₱{{ formatMoney(splitResult.totalUtilityShare) }}</div>
              <div class="mt-0.5 text-[11px] text-emerald-700">
                Electricity: ₱{{ formatMoney(splitResult.electricShare) }} + Water: ₱{{ formatMoney(splitResult.waterShare) }}
              </div>
            </div>
          </div>
        </section>

        <section class="space-y-4 rounded-xl border border-slate-200 bg-white p-5 shadow-sm lg:col-span-2">
          <div class="flex items-center justify-between border-b pb-2">
            <div class="flex items-center space-x-2 text-sm font-bold text-slate-800">
              <CreditCard class="h-4 w-4 text-emerald-600" />
              <span>Record a Payment</span>
            </div>
            <span class="rounded bg-slate-100 px-2 py-0.5 text-[11px] text-slate-600">
              {{ networkStore.isOnline ? 'Online mode' : 'Offline queue active' }}
            </span>
          </div>

          <div v-if="paymentFeedback" role="status" class="flex items-center justify-between rounded-lg border border-emerald-200 bg-emerald-50 p-3 text-xs text-emerald-800">
            <span>{{ paymentFeedback }}</span>
            <button @click="paymentFeedback = null" class="ml-2 font-bold underline">Dismiss</button>
          </div>
          <div v-if="paymentError" role="alert" class="rounded-lg border border-red-200 bg-red-50 p-3 text-xs text-red-800">
            {{ paymentError }}
          </div>

          <div class="grid grid-cols-1 gap-3 text-xs sm:grid-cols-3">
            <label class="font-medium text-slate-600">
              <span class="mb-1 block">Invoice</span>
              <select v-model="selectedInvoiceId" class="w-full rounded border border-slate-300 bg-white px-3 py-1.5">
                <option value="">Select an invoice</option>
                <option v-for="invoice in billingStore.invoices" :key="invoice.id" :value="invoice.id">
                  {{ invoice.invoiceNumber }} — ₱{{ formatMoney(outstandingBalance(invoice)) }} due
                </option>
              </select>
            </label>
            <label class="font-medium text-slate-600">
              <span class="mb-1 block">Payment amount (₱)</span>
              <input v-model.number="paymentAmount" type="number" min="0.01" :max="selectedBalance" step="0.01" class="w-full rounded border border-slate-300 px-3 py-1.5" />
              <span v-if="selectedInvoice" class="mt-1 block text-[11px] text-slate-500">Outstanding: ₱{{ formatMoney(selectedBalance) }}</span>
            </label>
            <label class="font-medium text-slate-600">
              <span class="mb-1 block">Method</span>
              <select v-model="paymentMethod" class="w-full rounded border border-slate-300 bg-white px-3 py-1.5">
                <option value="cash">Cash (on-site)</option>
                <option value="gcash">GCash</option>
                <option value="bank_transfer">Bank transfer</option>
              </select>
            </label>
          </div>
          <button
            @click="handlePayment"
            :disabled="!selectedInvoice || selectedBalance <= 0"
            class="rounded-lg bg-emerald-700 px-4 py-2 text-xs font-bold text-white transition-colors hover:bg-emerald-800 disabled:cursor-not-allowed disabled:opacity-50"
          >
            Record payment
          </button>
        </section>
      </div>

      <section id="rent-ledger" class="overflow-hidden rounded-xl border border-slate-200 bg-white shadow-sm">
        <div class="flex flex-wrap items-center justify-between gap-3 border-b border-slate-100 px-5 py-3">
          <div>
            <h2 class="text-sm font-bold text-slate-800">Monthly Invoices & Payment Ledger</h2>
            <p class="mt-1 text-xs text-slate-500">Paid and outstanding balances with recorded payment details.</p>
          </div>
          <div class="print-hidden flex gap-2">
            <button @click="downloadLedger" :disabled="billingStore.invoices.length === 0" class="inline-flex items-center gap-1.5 rounded border border-slate-300 px-3 py-1.5 text-xs font-semibold text-slate-700 hover:bg-slate-50 disabled:opacity-50">
              <Download class="h-3.5 w-3.5" />Download CSV
            </button>
            <button @click="printLedger" :disabled="billingStore.invoices.length === 0" class="inline-flex items-center gap-1.5 rounded border border-slate-300 px-3 py-1.5 text-xs font-semibold text-slate-700 hover:bg-slate-50 disabled:opacity-50">
              <Printer class="h-3.5 w-3.5" />Print
            </button>
          </div>
        </div>

        <div v-if="billingStore.loadError" role="alert" class="border-b border-amber-200 bg-amber-50 px-5 py-3 text-xs text-amber-800">
          {{ billingStore.loadError }}
        </div>

        <div class="overflow-x-auto">
          <table class="w-full min-w-[1180px] text-left text-xs">
            <thead class="border-b border-slate-200 bg-slate-50 uppercase tracking-wider text-slate-500">
              <tr>
                <th class="px-4 py-3">Invoice / Unit</th>
                <th class="px-4 py-3">Monthly charges</th>
                <th class="px-4 py-3">Total</th>
                <th class="px-4 py-3">Paid</th>
                <th class="px-4 py-3">Outstanding</th>
                <th class="px-4 py-3">Status</th>
                <th class="px-4 py-3">Due date</th>
                <th class="px-4 py-3">Payment history</th>
              </tr>
            </thead>
            <tbody class="divide-y divide-slate-100">
              <tr v-if="billingStore.loading">
                <td colspan="8" class="px-5 py-8 text-center text-slate-500">Loading billing records…</td>
              </tr>
              <tr v-else-if="billingStore.invoices.length === 0">
                <td colspan="8" class="px-5 py-8 text-center text-slate-400">No billing records found.</td>
              </tr>
              <tr v-for="invoice in billingStore.invoices" :key="invoice.id" class="align-top hover:bg-slate-50">
                <td class="px-4 py-3">
                  <div class="font-semibold text-slate-800">{{ invoice.invoiceNumber }}</div>
                  <div class="mt-1 text-slate-500">{{ invoice.property?.name || 'Property' }}</div>
                  <div class="text-slate-500">Unit {{ invoice.unit?.roomNumber || '—' }}</div>
                </td>
                <td class="px-4 py-3 text-slate-600">
                  <div>Rent: ₱{{ formatMoney(invoice.rentAmount) }}</div>
                  <div>Electricity: ₱{{ formatMoney(invoice.electricityAmount) }}</div>
                  <div>Water: ₱{{ formatMoney(invoice.waterAmount) }}</div>
                  <div>Other: ₱{{ formatMoney(invoice.otherFees) }}</div>
                </td>
                <td class="whitespace-nowrap px-4 py-3 font-bold text-slate-900">₱{{ formatMoney(invoice.totalAmount) }}</td>
                <td class="whitespace-nowrap px-4 py-3 text-emerald-700">₱{{ formatMoney(invoice.paidAmount) }}</td>
                <td class="whitespace-nowrap px-4 py-3 font-semibold text-slate-800">₱{{ formatMoney(outstandingBalance(invoice)) }}</td>
                <td class="px-4 py-3">
                  <span :class="[statusClass(invoice.status), 'whitespace-nowrap rounded-full px-2 py-0.5 text-[10px] font-bold uppercase tracking-wider']">
                    {{ invoice.status.replace('_', ' ') }}
                  </span>
                </td>
                <td class="whitespace-nowrap px-4 py-3 text-slate-500">{{ formatDate(invoice.dueDate) }}</td>
                <td class="px-4 py-3">
                  <ul v-if="invoice.payments?.length" class="space-y-2">
                    <li v-for="payment in invoice.payments" :key="payment.id" class="min-w-44 border-b border-slate-100 pb-2 last:border-0">
                      <div class="flex items-center gap-1.5 font-semibold text-slate-700">
                        <CheckCircle2 v-if="payment.isSynced" class="h-3.5 w-3.5 text-emerald-600" />
                        <Clock v-else class="h-3.5 w-3.5 text-amber-600" />
                        ₱{{ formatMoney(payment.amount) }} · {{ formatDate(payment.paymentDate) }}
                      </div>
                      <div class="mt-0.5 text-slate-500">
                        {{ payment.paymentMethod.replace('_', ' ') }} · {{ payment.isSynced ? 'Confirmed' : 'Awaiting sync' }}
                      </div>
                      <a v-if="payment.receiptUrl" :href="payment.receiptUrl" target="_blank" rel="noopener noreferrer" class="mt-0.5 inline-block text-msu-maroon underline">View receipt</a>
                    </li>
                  </ul>
                  <span v-else class="text-slate-400">No payments recorded</span>
                </td>
              </tr>
            </tbody>
          </table>
        </div>
      </section>
    </div>
  </AppLayout>
</template>

<style>
@media print {
  body * {
    visibility: hidden !important;
  }

  #rent-ledger,
  #rent-ledger * {
    visibility: visible !important;
  }

  #rent-ledger {
    position: absolute;
    inset: 0;
    width: 100%;
    overflow: visible !important;
    border: 0 !important;
    box-shadow: none !important;
  }

  #rent-ledger .print-hidden {
    display: none !important;
  }

  #rent-ledger table {
    min-width: 0 !important;
    font-size: 9pt;
  }
}
</style>
