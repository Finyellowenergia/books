<template>
  <div class="">
    <PageHeader :title="t`Point of Sale`">
      <slot>
        <Button
          class="bg-red-500 dark:bg-red-700"
          @click="toggleModal('ShiftClose')"
        >
          <span class="font-medium text-white">{{ t`Close POS Shift ` }}</span>
        </Button>
      </slot>
    </PageHeader>

    <OpenPOSShiftModal
      v-if="!isPosShiftOpen"
      :open-modal="!isPosShiftOpen"
      @toggle-modal="toggleModal"
    />

    <ClosePOSShiftModal
      :open-modal="openShiftCloseModal"
      @toggle-modal="toggleModal"
    />

    <PaymentModal
      :open-modal="openPaymentModal"
      @create-transaction="createTransaction"
      @toggle-modal="toggleModal"
      @set-cash-amount="setCashAmount"
      @set-transfer-amount="setTransferAmount"
      @set-transfer-ref-no="setTransferRefNo"
      @set-transfer-clearance-date="setTransferClearanceDate"
    />

    <div
      class="bg-gray-25 dark:bg-gray-875 gap-2 grid grid-cols-12 p-4"
      style="height: calc(100vh - var(--h-row-largest))"
    >
      <div
        class="
          bg-white
          dark:bg-gray-850
          border
          dark:border-gray-800
          col-span-5
          rounded-md
        "
      >
        <div class="rounded-md p-4 col-span-5">
          <div class="flex gap-x-2">
            <!-- Item Search -->
            <Link
              :class="
                fyo.singles.InventorySettings?.enableBarcodes
                  ? 'flex-shrink-0 w-2/3'
                  : 'w-full'
              "
              :df="{
                label: t`Search an Item`,
                fieldtype: 'Link',
                fieldname: 'item',
                target: 'Item',
              }"
              :border="true"
              :value="itemSearchTerm"
              @keyup.enter="
                async () => await addItem(await getItem(itemSearchTerm))
              "
              @change="(item: string) =>itemSearchTerm= item"
            />

            <Barcode
              v-if="fyo.singles.InventorySettings?.enableBarcodes"
              class="w-1/3"
              @item-selected="
                async (name: string) => {
                  await addItem(await getItem(name));
                }
              "
            />
          </div>

          <ItemsTable
            v-if="tableView"
            :items="items"
            :item-qty-map="itemQtyMap"
            @add-item="addItem"
          />
          <ItemsGrid
            v-else
            :items="items"
            :item-qty-map="itemQtyMap"
            @add-item="addItem"
          />
          <div
            class="bg-gray-100 p-2 fixed bottom-0 mb-7 rounded-md"
            @click="toggleView"
          >
            <FeatherIcon
              :name="tableView ? 'grid' : 'list'"
              class="w-6 h-6 text-black"
            />
          </div>
        </div>
      </div>

      <div class="col-span-7">
        <div class="flex flex-col gap-3" style="height: calc(100vh - 6rem)">
          <div
            class="
              bg-white
              dark:bg-gray-850
              border
              dark:border-gray-800
              grow
              h-full
              p-4
              rounded-md
            "
          >
            <!-- Customer Search -->
            <Link
              v-if="sinvDoc.fieldMap"
              class="flex-shrink-0"
              :border="true"
              :value="sinvDoc.party"
              :df="sinvDoc.fieldMap.party"
              @change="(value:string) => (sinvDoc.party = value)"
            />

            <SelectedItemTable />
          </div>

          <div
            class="
              bg-white
              dark:bg-gray-850
              border
              dark:border-gray-800
              p-4
              rounded-md
            "
          >
            <div class="w-full grid grid-cols-2 gap-y-2 gap-x-3">
              <div class="">
                <div class="grid grid-cols-2 gap-2">
                  <FloatingLabelFloatInput
                    :df="{
                      label: t`Total Quantity`,
                      fieldtype: 'Int',
                      fieldname: 'totalQuantity',
                      minvalue: 0,
                      maxvalue: 1000,
                    }"
                    size="large"
                    :value="totalQuantity"
                    :read-only="true"
                    :text-right="true"
                  />

                  <FloatingLabelCurrencyInput
                    :df="{
                      label: t`Add'l Discounts`,
                      fieldtype: 'Int',
                      fieldname: 'additionalDiscount',
                      minvalue: 0,
                    }"
                    size="large"
                    :value="additionalDiscounts"
                    :read-only="true"
                    :text-right="true"
                    @change="(amount:Money)=> additionalDiscounts= amount"
                  />
                </div>

                <div class="mt-4 grid grid-cols-2 gap-2">
                  <FloatingLabelCurrencyInput
                    :df="{
                      label: t`Item Discounts`,
                      fieldtype: 'Currency',
                      fieldname: 'itemDiscounts',
                    }"
                    size="large"
                    :value="itemDiscounts"
                    :read-only="true"
                    :text-right="true"
                  />
                  <FloatingLabelCurrencyInput
                    v-if="sinvDoc.fieldMap"
                    :df="sinvDoc.fieldMap.grandTotal"
                    size="large"
                    :value="sinvDoc.grandTotal"
                    :read-only="true"
                    :text-right="true"
                  />
                </div>
              </div>

              <div class="">
                <Button
                  class="w-full bg-red-500 dark:bg-red-700 py-6"
                  :disabled="!sinvDoc.items?.length"
                  @click="clearValues"
                >
                  <slot>
                    <p class="uppercase text-lg text-white font-semibold">
                      {{ t`Cancel` }}
                    </p>
                  </slot>
                </Button>

                <Button
                  class="mt-4 w-full bg-green-500 dark:bg-green-700 py-6"
                  :disabled="disablePayButton"
                  @click="toggleModal('Payment', true)"
                >
                  <slot>
                    <p class="uppercase text-lg text-white font-semibold">
                      {{ t`Pay` }}
                    </p>
                  </slot>
                </Button>
              </div>
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script lang="ts">
import Button from 'src/components/Button.vue';
import ClosePOSShiftModal from './ClosePOSShiftModal.vue';
import FloatingLabelCurrencyInput from 'src/components/POS/FloatingLabelCurrencyInput.vue';
import FloatingLabelFloatInput from 'src/components/POS/FloatingLabelFloatInput.vue';
import ItemsTable from 'src/components/POS/ItemsTable.vue';
import Link from 'src/components/Controls/Link.vue';
import OpenPOSShiftModal from './OpenPOSShiftModal.vue';
import PageHeader from 'src/components/PageHeader.vue';
import PaymentModal from './PaymentModal.vue';
import SelectedItemTable from 'src/components/POS/SelectedItemTable.vue';
import { computed, defineComponent } from 'vue';
import { fyo } from 'src/initFyo';
import { routeTo, toggleSidebar } from 'src/utils/ui';
import { ModelNameEnum } from 'models/types';
import { SalesInvoice } from 'models/baseModels/SalesInvoice/SalesInvoice';
import ItemsGrid from 'src/components/POS/ItemsGrid.vue';
import { t } from 'fyo';
import {
  ItemQtyMap,
  ItemSerialNumbers,
  POSItem,
} from 'src/components/POS/types';
import { Item } from 'models/baseModels/Item/Item';
import { ModalName } from 'src/components/POS/types';
import { Money } from 'pesa';
import { Payment } from 'models/baseModels/Payment/Payment';
import { SalesInvoiceItem } from 'models/baseModels/SalesInvoiceItem/SalesInvoiceItem';
import { Shipment } from 'models/inventory/Shipment';
import { showToast } from 'src/utils/interactive';
import {
  getItem,
  getItemDiscounts,
  getItemQtyMap,
  getTotalQuantity,
  getTotalTaxedAmount,
  validateIsPosSettingsSet,
  validateShipment,
  validateSinv,
} from 'src/utils/pos';
import Barcode from 'src/components/Controls/Barcode.vue';
import { getPricingRule } from 'models/helpers';

export default defineComponent({
  name: 'POS',
  components: {
    Button,
    ClosePOSShiftModal,
    FloatingLabelCurrencyInput,
    FloatingLabelFloatInput,
    ItemsTable,
    ItemsGrid,
    Link,
    OpenPOSShiftModal,
    PageHeader,
    PaymentModal,
    SelectedItemTable,
    Barcode,
  },
  provide() {
    return {
      cashAmount: computed(() => this.cashAmount),
      doc: computed(() => this.sinvDoc),
      isDiscountingEnabled: computed(() => this.isDiscountingEnabled),
      itemDiscounts: computed(() => this.itemDiscounts),
      itemQtyMap: computed(() => this.itemQtyMap),
      itemSerialNumbers: computed(() => this.itemSerialNumbers),
      sinvDoc: computed(() => this.sinvDoc),
      totalTaxedAmount: computed(() => this.totalTaxedAmount),
      transferAmount: computed(() => this.transferAmount),
      transferClearanceDate: computed(() => this.transferClearanceDate),
      transferRefNo: computed(() => this.transferRefNo),
    };
  },
  data() {
    return {
      items: [] as POSItem[],

      tableView: true,

      isItemsSeeded: false,
      openPaymentModal: false,
      openShiftCloseModal: false,
      openShiftOpenModal: false,

      additionalDiscounts: fyo.pesa(0),
      cashAmount: fyo.pesa(0),
      itemDiscounts: fyo.pesa(0),
      totalTaxedAmount: fyo.pesa(0),
      transferAmount: fyo.pesa(0),

      totalQuantity: 0,

      defaultCustomer: undefined as string | undefined,
      itemSearchTerm: '',
      transferRefNo: undefined as string | undefined,

      transferClearanceDate: undefined as Date | undefined,

      itemQtyMap: {} as ItemQtyMap,
      itemSerialNumbers: {} as ItemSerialNumbers,
      paymentDoc: {} as Payment,
      sinvDoc: {} as SalesInvoice,
    };
  },
  computed: {
    defaultPOSCashAccount: () =>
      fyo.singles.POSSettings?.cashAccount ?? undefined,
    isDiscountingEnabled(): boolean {
      return !!fyo.singles.AccountingSettings?.enableDiscounting;
    },
    isPosShiftOpen: () => !!fyo.singles.POSShift?.isShiftOpen,
    isPaymentAmountSet(): boolean {
      if (this.sinvDoc.grandTotal?.isZero()) {
        return true;
      }

      if (this.cashAmount.isZero() && this.transferAmount.isZero()) {
        return false;
      }
      return true;
    },
    disablePayButton(): boolean {
      if (!this.sinvDoc.items?.length) {
        return true;
      }

      if (!this.sinvDoc.party) {
        return true;
      }
      return false;
    },
  },
  watch: {
    sinvDoc: {
      handler() {
        this.updateValues();
      },
      deep: true,
    },
  },
  async mounted() {
    await this.setItems();
  },
  async activated() {
    toggleSidebar(false);
    validateIsPosSettingsSet(fyo);
    this.setSinvDoc();
    this.setDefaultCustomer();
    await this.setItemQtyMap();
    await this.setItems();
  },
  deactivated() {
    toggleSidebar(true);
  },
  methods: {
    async setItems() {
      const items = (await fyo.db.getAll(ModelNameEnum.Item, {
        fields: [],
        filters: { trackItem: true },
      })) as Item[];

      this.items = [] as POSItem[];
      for (const item of items) {
        let availableQty = 0;

        if (!!this.itemQtyMap[item.name as string]) {
          availableQty = this.itemQtyMap[item.name as string].availableQty;
        }

        if (!item.name) {
          return;
        }

        this.items.push({
          availableQty,
          name: item.name,
          image: item?.image as string,
          rate: item.rate as Money,
          unit: item.unit as string,
          hasBatch: !!item.hasBatch,
          hasSerialNumber: !!item.hasSerialNumber,
        });
      }
    },
    toggleView() {
      this.tableView = !this.tableView;
    },
    setCashAmount(amount: Money) {
      this.cashAmount = amount;
    },
    setDefaultCustomer() {
      this.defaultCustomer = this.fyo.singles.Defaults?.posCustomer ?? '';
      this.sinvDoc.party = this.defaultCustomer;
    },
    setItemDiscounts() {
      this.itemDiscounts = getItemDiscounts(
        this.sinvDoc.items as SalesInvoiceItem[]
      );
    },
    async setItemQtyMap() {
      this.itemQtyMap = await getItemQtyMap();
    },
    setSinvDoc() {
      this.sinvDoc = this.fyo.doc.getNewDoc(ModelNameEnum.SalesInvoice, {
        account: 'Debtors',
        party: this.sinvDoc.party ?? this.defaultCustomer,
        isPOS: true,
      }) as SalesInvoice;
    },
    setTotalQuantity() {
      this.totalQuantity = getTotalQuantity(
        this.sinvDoc.items as SalesInvoiceItem[]
      );
    },
    setTotalTaxedAmount() {
      this.totalTaxedAmount = getTotalTaxedAmount(this.sinvDoc as SalesInvoice);
    },
    setTransferAmount(amount: Money = fyo.pesa(0)) {
      this.transferAmount = amount;
    },
    setTransferClearanceDate(date: Date) {
      this.transferClearanceDate = date;
    },
    setTransferRefNo(ref: string) {
      this.transferRefNo = ref;
    },
    removeFreeItems() {
      if (!this.sinvDoc || !this.sinvDoc.items) {
        return;
      }

      if (!!this.sinvDoc.isPricingRuleApplied) {
        return;
      }

      for (const item of this.sinvDoc.items) {
        if (item.isFreeItem) {
          this.sinvDoc.items = this.sinvDoc.items?.filter(
            (invoiceItem) => invoiceItem.name !== item.name
          );
        }
      }
    },

    async addItem(item: POSItem | Item | undefined) {
      // eslint-disable-next-line @typescript-eslint/no-floating-promises
      await this.sinvDoc.runFormulas();

      if (!item) {
        return;
      }

      if (
        !this.itemQtyMap[item.name as string] ||
        this.itemQtyMap[item.name as string].availableQty === 0
      ) {
        showToast({
          type: 'error',
          message: t`Item ${item.name as string} has Zero Quantity`,
          duration: 'short',
        });
        return;
      }

      const existingItems =
        this.sinvDoc.items?.filter(
          (invoiceItem) =>
            invoiceItem.item === item.name && !invoiceItem.isFreeItem
        ) ?? [];

      if (item.hasBatch) {
        for (const invItem of existingItems) {
          const itemQty = invItem.quantity ?? 0;
          const qtyInBatch =
            this.itemQtyMap[invItem.item as string][invItem.batch as string] ??
            0;

          if (itemQty < qtyInBatch) {
            invItem.quantity = (invItem.quantity as number) + 1;
            invItem.rate = item.rate as Money;

            await this.applyPricingRule();
            await this.sinvDoc.runFormulas();

            return;
          }
        }

        try {
          await this.sinvDoc.append('items', {
            rate: item.rate as Money,
            item: item.name,
          });
        } catch (error) {
          showToast({
            type: 'error',
            message: t`${error as string}`,
          });
        }

        return;
      }

      if (existingItems.length) {
        existingItems[0].rate = item.rate as Money;
        existingItems[0].quantity = (existingItems[0].quantity as number) + 1;
        await this.applyPricingRule();
        await this.sinvDoc.runFormulas();
        return;
      }

      await this.sinvDoc.append('items', {
        rate: item.rate as Money,
        item: item.name,
      });

      await this.applyPricingRule();
      await this.sinvDoc.runFormulas();
    },
    async createTransaction(shouldPrint = false) {
      try {
        await this.validate();
        await this.submitSinvDoc(shouldPrint);
        await this.makePayment();
        await this.makeStockTransfer();
        await this.afterTransaction();
      } catch (error) {
        showToast({
          type: 'error',
          message: t`${error as string}`,
        });
      }
    },
    async makePayment() {
      this.paymentDoc = this.sinvDoc.getPayment() as Payment;
      const paymentMethod = this.cashAmount.isZero() ? 'Transfer' : 'Cash';
      await this.paymentDoc.set('paymentMethod', paymentMethod);

      if (paymentMethod === 'Transfer') {
        await this.paymentDoc.setMultiple({
          amount: this.transferAmount as Money,
          referenceId: this.transferRefNo,
          clearanceDate: this.transferClearanceDate,
        });
      }

      if (paymentMethod === 'Cash') {
        await this.paymentDoc.setMultiple({
          paymentAccount: this.defaultPOSCashAccount,
          amount: this.cashAmount as Money,
        });
      }

      this.paymentDoc.once('afterSubmit', () => {
        showToast({
          type: 'success',
          message: t`Payment ${this.paymentDoc.name as string} is Saved`,
          duration: 'short',
        });
      });

      try {
        await this.paymentDoc?.sync();
        await this.paymentDoc?.submit();
      } catch (error) {
        return showToast({
          type: 'error',
          message: t`${error as string}`,
        });
      }
    },
    async makeStockTransfer() {
      const shipmentDoc = (await this.sinvDoc.getStockTransfer()) as Shipment;
      if (!shipmentDoc.items) {
        return;
      }

      for (const item of shipmentDoc.items) {
        item.location = fyo.singles.POSSettings?.inventory;
        item.serialNumber =
          this.itemSerialNumbers[item.item as string] ?? undefined;
      }

      shipmentDoc.once('afterSubmit', () => {
        showToast({
          type: 'success',
          message: t`Shipment ${shipmentDoc.name as string} is Submitted`,
          duration: 'short',
        });
      });

      try {
        await shipmentDoc.sync();
        await shipmentDoc.submit();
      } catch (error) {
        return showToast({
          type: 'error',
          message: t`${error as string}`,
        });
      }
    },
    async submitSinvDoc(shouldPrint: boolean) {
      this.sinvDoc.once('afterSubmit', async () => {
        showToast({
          type: 'success',
          message: t`Sales Invoice ${this.sinvDoc.name as string} is Submitted`,
          duration: 'short',
        });

        if (shouldPrint) {
          await routeTo(
            // eslint-disable-next-line @typescript-eslint/restrict-template-expressions
            `/print/${this.sinvDoc.schemaName}/${this.sinvDoc.name}`
          );
        }
      });

      try {
        await this.validate();
        await this.sinvDoc.runFormulas();
        await this.sinvDoc.sync();
        await this.sinvDoc.submit();
      } catch (error) {
        return showToast({
          type: 'error',
          message: t`${error as string}`,
        });
      }
    },

    async afterTransaction() {
      await this.setItemQtyMap();
      this.clearValues();
      this.setSinvDoc();
      this.toggleModal('Payment', false);
    },
    clearValues() {
      this.setSinvDoc();
      this.itemSerialNumbers = {};

      this.cashAmount = fyo.pesa(0);
      this.transferAmount = fyo.pesa(0);
    },
    toggleModal(modal: ModalName, value?: boolean) {
      if (value) {
        return (this[`open${modal}Modal`] = value);
      }
      return (this[`open${modal}Modal`] = !this[`open${modal}Modal`]);
    },
    updateValues() {
      this.setTotalQuantity();
      this.setItemDiscounts();
      this.setTotalTaxedAmount();
    },
    async validate() {
      validateSinv(this.sinvDoc as SalesInvoice, this.itemQtyMap);
      await validateShipment(this.itemSerialNumbers);
    },
    async applyPricingRule() {
      const hasPricingRules = await getPricingRule(
        this.sinvDoc as SalesInvoice
      );

      if (!hasPricingRules || !hasPricingRules.length) {
        this.sinvDoc.pricingRuleDetail = undefined;
        this.sinvDoc.isPricingRuleApplied = false;
        this.removeFreeItems();
        return;
      }

      setTimeout(async () => {
        const appliedPricingRuleCount = this.sinvDoc?.items?.filter(
          (val) => val.isFreeItem
        ).length;

        if (appliedPricingRuleCount !== hasPricingRules?.length) {
          await this.sinvDoc.appendPricingRuleDetail(hasPricingRules);
          await this.sinvDoc.applyProductDiscount();
        }
      }, 1);
    },

    getItem,
  },
});
</script>
