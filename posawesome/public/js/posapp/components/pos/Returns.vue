<template>
  <v-row justify="center">
    <v-dialog v-model="invoicesDialog" max-width="800px" min-width="800px">
      <v-card>
        <v-card-title>
          <span class="headline primary--text">{{
            __('Select Return Invoice')
          }}</span>
        </v-card-title>
        <v-container>
          <v-row class="mb-4">
            <v-col cols="3">
              <v-text-field
                color="primary"
                :label="frappe._('Invoice ID *')"
                background-color="white"
                hide-details
                v-model="invoice_name"
                dense
                clearable
                @keydown.enter="search_invoices"
            ></v-text-field>
            </v-col>
            <v-col cols="3">
              <v-text-field
                color="primary"
                :label="frappe._('Customer Name *')"
                background-color="white"
                hide-details
                v-model="customer_name"
                dense
                clearable
                @keydown.enter="search_invoices"
              ></v-text-field>
            </v-col>
            <v-col cols="3">
              <v-text-field
                color="primary"
                :label="frappe._('Full FS Account No.*')"
                background-color="white"
                hide-details
                v-model="custom_fs_account_number"
                dense
                clearable
                @keydown.enter="search_invoices"
              ></v-text-field>
            </v-col>
            <v-col cols="2">
              <v-text-field
                color="secondary"
                :label="frappe._('Item Code')"
                background-color="white"
                hide-details
                v-model="item_code"
                dense
                clearable
                @keydown.enter="search_invoices"
              ></v-text-field>
            </v-col>
            <!--v-col cols="2">
              <v-btn
                text
                class="ml-2"
                color="primary"
                dark
                @click="search_invoices"
              >{{ __('Search') }}</v-btn>
            </v-col-->
          </v-row>
          <v-row>
            <v-col cols="12" class="pa-1" v-if="dialog_data">
              <template>
                <v-data-table
                  :headers="headers"
                  :items="dialog_data"
                  item-key="name"
                  class="elevation-1"
                  :single-select="singleSelect"
                  show-select
                  v-model="selected"
                >
                  <template v-slot:item.grand_total="{ item }">
                    {{ currencySymbol(item.currency) }}
                    {{ formtCurrency(item.grand_total) }}</template
                  >
                </v-data-table>
              </template>
            </v-col>
          </v-row>
        </v-container>
        <v-card-actions class="mt-4">
          <v-spacer></v-spacer>
          <v-btn
            text
            class="ml-2"
            color="primary"
            dark
            @click="search_invoices"
          >{{ __('Search') }}
          </v-btn>
          <v-btn color="error mx-2" dark @click="close_dialog">Close</v-btn>
          <v-btn
            v-if="selected.length"
            color="success"
            dark
            @click="submit_dialog"
            >{{ __('Select') }}</v-btn
          >
        </v-card-actions>
      </v-card>
    </v-dialog>
  </v-row>
</template>

<script>
import { evntBus } from '../../bus';
import format from '../../format';
export default {
  mixins: [format],
  data: () => ({
    invoicesDialog: false,
    singleSelect: true,
    selected: [],
    dialog_data: '',
    company: '',
    invoice_name: '',
    customer_name: '', // for return search
    custom_fs_account_number: '', // for return search
    item_code: '', // for return search
    headers: [
      {
        text: __('Customer'),
        value: 'customer_name',
        align: 'start',
        sortable: true,
      },
      {
        text: __('Date'),
        align: 'start',
        sortable: true,
        value: 'posting_date',
      },
      {
        text: __('Invoice'),
        value: 'name',
        align: 'start',
        sortable: true,
      },
      {
        text: __('Amount'),
        value: 'grand_total',
        align: 'end',
        sortable: false,
      },
    ],
  }),
  watch: {},
  methods: {
    close_dialog() {
      this.invoicesDialog = false;
    },
    search_invoices_by_enter(e) {
      if (e.keyCode === 13) {
        this.search_invoices();
      }
    },
    search_invoices() {
      const vm = this;
      frappe.call({
        method: 'posawesome.posawesome.api.posapp.search_invoices_for_return',
        args: {
          invoice_name: vm.invoice_name,
          customer_name: vm.customer_name,
          custom_fs_account_number: vm.custom_fs_account_number,
          item_code: vm.item_code,
          company: vm.company,
        },
        async: false,
        callback: function (r) {
          if (r.message) {
            if (r.message instanceof Array) { // check whether there is a result or an error message
              vm.dialog_data = r.message;
            }
            else {
              evntBus.$emit('show_mesage', {
                text: r.message,
                color: 'warning',
              });
            }
          }
        },
      });
    },
    submit_dialog() {
      if (this.selected.length > 0) {
        const return_doc = this.selected[0];
        const invoice_doc = {};
        const items = [];
        return_doc.items.forEach((item) => {
          const new_item = { ...item };
          new_item.qty = item.qty * -1;
          new_item.stock_qty = item.stock_qty * -1;
          new_item.amount = item.amount * -1;
          items.push(new_item);
        });
        invoice_doc.items = items;
        invoice_doc.is_return = 1;
        invoice_doc.return_against = return_doc.name;
        invoice_doc.customer = return_doc.customer;
        const data = { invoice_doc, return_doc };
        evntBus.$emit('load_return_invoice', data);
        this.invoicesDialog = false;
      }
    },
  },
  created: function () {
    evntBus.$on('open_returns', (data) => {
      this.invoicesDialog = true;
      this.company = data;
      this.invoice_name = '';
      this.customer_name = '';
      this.custom_fs_account_number = '';
      this.item_code = '';
      this.dialog_data = '';
      this.selected = [];
    });
  },
};
</script>
