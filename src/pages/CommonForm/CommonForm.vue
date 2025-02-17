<template>
  <FormContainer>
    <template #header-left v-if="hasDoc">
      <StatusBadge :status="status" class="h-8" />
      <Barcode
        class="h-8"
        v-if="canShowBarcode"
        @item-selected="(name:string) => {
          // @ts-ignore
          doc?.addItem(name);
        }"
      />
      <ExchangeRate
        v-if="hasDoc && doc.isMultiCurrency"
        :disabled="doc?.isSubmitted || doc?.isCancelled"
        :from-currency="fromCurrency"
        :to-currency="toCurrency"
        :exchange-rate="exchangeRate"
        @change="
          async (exchangeRate: number) =>
            await doc.set('exchangeRate', exchangeRate)
        "
      />
    </template>
    <template #header v-if="hasDoc">
      <Button
        v-if="canShowLinks"
        :icon="true"
        @click="showLinks = true"
        :title="t`View linked entries`"
      >
        <feather-icon name="link" class="w-4 h-4"></feather-icon>
      </Button>
      <Button
        v-if="canPrint"
        ref="printButton"
        :icon="true"
        @click="routeTo(`/print/${doc.schemaName}/${doc.name}`)"
        :title="t`Open Print View`"
      >
        <feather-icon name="printer" class="w-4 h-4"></feather-icon>
      </Button>
      <DropdownWithActions
        v-for="group of groupedActions"
        :key="group.label"
        :type="group.type"
        :actions="group.actions"
      >
        <p v-if="group.group">
          {{ group.group }}
        </p>
        <feather-icon v-else name="more-horizontal" class="w-4 h-4" />
      </DropdownWithActions>
      <Button v-if="doc?.canSave" type="primary" @click="sync">
        {{ t`Save` }}
      </Button>
      <Button v-else-if="doc?.canSubmit" type="primary" @click="submit">{{
        t`Submit`
      }}</Button>
    </template>
    <template #body>
      <FormHeader
        :form-title="title"
        :form-sub-title="schema.label"
        class="sticky top-0 bg-white border-b"
      >
      </FormHeader>

      <!-- Section Container -->
      <div v-if="hasDoc" class="overflow-auto custom-scroll">
        <CommonFormSection
          v-for="([name, fields], idx) in activeGroup.entries()"
          @editrow="(doc: Doc) => toggleQuickEditDoc(doc)"
          :key="name + idx"
          ref="section"
          class="p-4"
          :class="idx !== 0 && activeGroup.size > 1 ? 'border-t' : ''"
          :show-title="activeGroup.size > 1 && name !== t`Default`"
          :title="name"
          :fields="fields"
          :doc="doc"
          :errors="errors"
          @value-change="onValueChange"
          @row-change="updateGroupedFields"
        />
      </div>

      <!-- Tab Bar -->
      <div
        class="
          mt-auto
          px-4
          pb-4
          flex
          gap-8
          border-t
          flex-shrink-0
          sticky
          bottom-0
          bg-white
        "
        v-if="groupedFields && groupedFields.size > 1"
      >
        <div
          v-for="key of groupedFields.keys()"
          :key="key"
          @click="activeTab = key"
          class="text-sm cursor-pointer"
          :class="
            key === activeTab
              ? 'text-blue-500 font-semibold border-t-2 border-blue-500'
              : ''
          "
          :style="{
            paddingTop: key === activeTab ? 'calc(1rem - 2px)' : '1rem',
          }"
        >
          {{ key }}
        </div>
      </div>
    </template>
    <template #quickedit>
      <Transition name="quickedit">
        <QuickEditForm
          v-if="hasQeDoc"
          :name="qeDoc.name"
          :show-name="false"
          :show-save="false"
          :source-doc="qeDoc"
          :schema-name="qeDoc.schemaName"
          :white="true"
          :route-back="false"
          :load-on-close="false"
          @close="() => toggleQuickEditDoc(null)"
        />
      </Transition>
      <Transition name="quickedit">
        <LinkedEntries
          v-if="showLinks && !hasQeDoc"
          :doc="doc"
          @close="showLinks = false"
        />
      </Transition>
    </template>
  </FormContainer>
</template>
<script lang="ts">
import { DocValue } from 'fyo/core/types';
import { Doc } from 'fyo/model/doc';
import { ValidationError } from 'fyo/utils/errors';
import { getDocStatus } from 'models/helpers';
import { ModelNameEnum } from 'models/types';
import { Field, Schema } from 'schemas/types';
import Button from 'src/components/Button.vue';
import DropdownWithActions from 'src/components/DropdownWithActions.vue';
import FormContainer from 'src/components/FormContainer.vue';
import FormHeader from 'src/components/FormHeader.vue';
import StatusBadge from 'src/components/StatusBadge.vue';
import { getErrorMessage } from 'src/utils';
import { docsPathMap } from 'src/utils/misc';
import { docsPathRef } from 'src/utils/refs';
import { ActionGroup, DocRef, UIGroupedFields } from 'src/utils/types';
import {
  commonDocSubmit,
  commonDocSync,
  getDocFromNameIfExistsElseNew,
  getFieldsGroupedByTabAndSection,
  getGroupedActionsForDoc,
  isPrintable,
  routeTo,
} from 'src/utils/ui';
import { computed, defineComponent, nextTick } from 'vue';
import QuickEditForm from '../QuickEditForm.vue';
import CommonFormSection from './CommonFormSection.vue';
import { inject } from 'vue';
import { shortcutsKey } from 'src/utils/injectionKeys';
import { ref } from 'vue';
import { useDocShortcuts } from 'src/utils/vueUtils';
import Barcode from 'src/components/Controls/Barcode.vue';
import { DEFAULT_CURRENCY } from 'fyo/utils/consts';
import ExchangeRate from 'src/components/Controls/ExchangeRate.vue';
import LinkedEntries from './LinkedEntries.vue';

export default defineComponent({
  props: {
    name: { type: String, default: '' },
    schemaName: { type: String, default: ModelNameEnum.SalesInvoice },
  },
  setup() {
    const shortcuts = inject(shortcutsKey);
    const docOrNull = ref(null) as DocRef;
    let context = 'CommonForm';
    if (shortcuts) {
      context = useDocShortcuts(shortcuts, docOrNull, 'CommonForm', true);
    }

    return {
      docOrNull,
      shortcuts,
      context,
      printButton: ref<InstanceType<typeof Button> | null>(null),
    };
  },
  provide() {
    return {
      doc: computed(() => this.docOrNull),
    };
  },
  data() {
    return {
      errors: {},
      activeTab: this.t`Default`,
      groupedFields: null,
      quickEditDoc: null,
      isPrintable: false,
      showLinks: false,
    } as {
      errors: Record<string, string>;
      activeTab: string;
      groupedFields: null | UIGroupedFields;
      quickEditDoc: null | Doc;
      isPrintable: boolean;
      showLinks: boolean;
    };
  },
  async mounted() {
    if (this.fyo.store.isDevelopment) {
      // @ts-ignore
      window.cf = this;
    }

    await this.setDoc();
    this.updateGroupedFields();
    if (this.groupedFields) {
      this.activeTab = [...this.groupedFields.keys()][0];
    }
    this.isPrintable = await isPrintable(this.schemaName);
  },
  activated(): void {
    docsPathRef.value = docsPathMap[this.schemaName] ?? '';
    this.shortcuts?.pmod.set(this.context, ['KeyP'], () => {
      if (!this.canPrint) {
        return;
      }

      this.printButton?.$el.click();
    });
    this.shortcuts?.pmod.set(this.context, ['KeyL'], () => {
      if (!this.canShowLinks && !this.showLinks) {
        return;
      }

      this.showLinks = !this.showLinks;
    });
  },
  deactivated(): void {
    docsPathRef.value = '';
    this.showLinks = false;
  },
  computed: {
    canShowBarcode(): boolean {
      if (!this.fyo.singles.InventorySettings?.enableBarcodes) {
        return false;
      }

      if (!this.hasDoc) {
        return false;
      }

      if (this.doc.isSubmitted || this.doc.isCancelled) {
        return false;
      }

      // @ts-ignore
      return typeof this.doc?.addItem === 'function';
    },
    exchangeRate(): number {
      if (!this.hasDoc || typeof this.doc.exchangeRate !== 'number') {
        return 1;
      }

      return this.doc.exchangeRate;
    },
    fromCurrency(): string {
      const currency = this.doc?.currency;
      if (typeof currency !== 'string') {
        return this.toCurrency;
      }

      return currency;
    },
    toCurrency(): string {
      const currency = this.fyo.singles.SystemSettings?.currency;
      if (typeof currency !== 'string') {
        return DEFAULT_CURRENCY;
      }

      return currency;
    },
    canPrint(): boolean {
      if (!this.hasDoc) {
        return false;
      }

      return !this.doc.isCancelled && !this.doc.dirty && this.isPrintable;
    },
    canShowLinks(): boolean {
      if (!this.hasDoc || this.hasQeDoc) {
        return false;
      }

      if (this.doc.schema.isSubmittable && !this.doc.isSubmitted) {
        return false;
      }

      return this.doc.inserted;
    },
    hasDoc(): boolean {
      return this.docOrNull instanceof Doc;
    },
    hasQeDoc(): boolean {
      return this.quickEditDoc instanceof Doc;
    },
    status(): string {
      if (!this.hasDoc) {
        return '';
      }

      return getDocStatus(this.doc);
    },
    doc(): Doc {
      const doc = this.docOrNull as Doc | null;
      if (!doc) {
        throw new ValidationError(
          this.t`Doc ${this.schema.label} ${this.name} not set`
        );
      }
      return doc;
    },
    qeDoc(): Doc {
      const doc = this.quickEditDoc as Doc | null;
      if (!doc) {
        throw new ValidationError(
          this.t`Doc ${this.schema.label} ${this.name} not set`
        );
      }
      return doc;
    },
    title(): string {
      if (this.schema.isSubmittable && this.docOrNull?.notInserted) {
        return this.t`New Entry`;
      }

      return this.docOrNull?.name! ?? this.t`New Entry`;
    },
    schema(): Schema {
      const schema = this.fyo.schemaMap[this.schemaName];
      if (!schema) {
        throw new ValidationError(`no schema found with ${this.schemaName}`);
      }

      return schema;
    },
    activeGroup(): Map<string, Field[]> {
      if (!this.groupedFields) {
        return new Map();
      }

      const group = this.groupedFields.get(this.activeTab);
      if (!group) {
        throw new ValidationError(
          `Tab group ${this.activeTab} has no value set`
        );
      }

      return group;
    },
    groupedActions(): ActionGroup[] {
      if (!this.hasDoc) {
        return [];
      }

      return getGroupedActionsForDoc(this.doc);
    },
  },
  methods: {
    routeTo,
    updateGroupedFields(): void {
      if (!this.hasDoc) {
        return;
      }

      this.groupedFields = getFieldsGroupedByTabAndSection(
        this.schema,
        this.doc
      );
    },
    async sync(useDialog?: boolean) {
      if (await commonDocSync(this.doc, useDialog)) {
        this.updateGroupedFields();
      }
    },
    async submit() {
      if (await commonDocSubmit(this.doc)) {
        this.updateGroupedFields();
      }
    },
    async setDoc() {
      if (this.hasDoc) {
        return;
      }

      this.docOrNull = await getDocFromNameIfExistsElseNew(
        this.schemaName,
        this.name
      );
    },
    async toggleQuickEditDoc(doc: Doc | null) {
      if (this.quickEditDoc && doc) {
        this.quickEditDoc = null;
        await nextTick();
      }

      if (doc && this.showLinks) {
        this.showLinks = false;
        await nextTick();
      }

      this.quickEditDoc = doc;
    },
    async onValueChange(field: Field, value: DocValue) {
      const { fieldname } = field;
      delete this.errors[fieldname];

      try {
        await this.doc.set(fieldname, value);
      } catch (err) {
        if (!(err instanceof Error)) {
          return;
        }

        this.errors[fieldname] = getErrorMessage(err, this.doc);
      }

      this.updateGroupedFields();
    },
  },
  components: {
    FormContainer,
    FormHeader,
    CommonFormSection,
    StatusBadge,
    Button,
    DropdownWithActions,
    QuickEditForm,
    Barcode,
    ExchangeRate,
    LinkedEntries,
  },
});
</script>
