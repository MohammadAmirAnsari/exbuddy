<template>
    <div>
        <label class="block mb-2.5 text-sm font-medium text-heading">{{ label }}</label>

        <div class="relative group">
            <input type="text" :value="modelValue" @input="updateValue($event.target.value)" :placeholder="placeholder"
                required
                class="bg-neutral-secondary-medium border border-default-medium text-heading text-sm rounded-base focus:ring-brand focus:border-brand block w-full px-3 py-2.5 shadow-xs" />

            <ul
                class="hidden group-focus-within:block absolute z-50 w-full mt-1 max-h-60 overflow-y-auto bg-neutral-secondary-medium border border-default-medium rounded-base shadow-lg divide-y divide-default-medium/20">

                <li v-for="item in options.filter(o => o.toLowerCase().includes((modelValue || '').toLowerCase()))"
                    :key="item" @mousedown="selectItem(item)"
                    class="px-4 py-2.5 text-sm text-heading hover:bg-brand hover:text-white cursor-pointer transition-colors duration-150">
                    {{ item }}
                </li>

                <li v-if="options.filter(o => o.toLowerCase().includes((modelValue || '').toLowerCase())).length === 0"
                    class="px-4 py-2.5 text-sm text-body italic text-center">
                    No match found
                </li>
            </ul>
        </div>
    </div>
</template>

<script>
export default {
    name: 'SearchableSelect',
    props: {
        label: { type: String, required: true },
        placeholder: { type: String, default: 'Type to search...' },
        options: { type: Array, required: true },
        modelValue: { type: String, required: true }
    },
    emits: ['update:modelValue'],
    methods: {
        updateValue(val) {
            this.$emit('update:modelValue', val);
        },
        selectItem(val) {
            this.$emit('update:modelValue', val);
        }
    }
}
</script>
