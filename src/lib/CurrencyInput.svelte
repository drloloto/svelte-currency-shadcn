<script lang="ts">
import { onMount } from 'svelte';
import type { HTMLInputAttributes } from 'svelte/elements';
import { type InputEvents, cn } from './index.js';

type CustomProps = {
	value?: any;
	class?: string;
	onValueChange?: Callback;
	locale?: string;
	currency?: string;
	name?: string;
	required?: boolean;
	disabled?: boolean;
	placeholder?: string | number | null;
	autocomplete?: string | null | undefined;
	isNegativeAllowed?: boolean;
	isZeroNullish?: boolean;
	fractionDigits?: number;
	inputClasses?: InputClasses | null;
};
interface InputClasses {
	wrapper?: string | undefined;
	unformatted?: string | undefined;
	formatted?: string | undefined;
	formattedPositive?: string | undefined;
	formattedNegative?: string | undefined;
	formattedZero?: string | undefined;
}

type $$Props = CustomProps & Omit<HTMLInputAttributes, keyof CustomProps>;
type $$Events = InputEvents;
let className: $$Props['class'] = undefined;
export { className as class };
export let readonly: $$Props['readonly'] = undefined;

const DEFAULT_LOCALE = 'en-US';
const DEFAULT_CURRENCY = 'USD';
const DEFAULT_NAME = 'total';
const DEFAULT_VALUE: number = 0;
const DEFAULT_FRACTION_DIGITS = 2;

const DEFAULT_CLASS_UNFORMATTED = '';
const DEFAULT_CLASS_FORMATTED =
	'flex h-10 w-full rounded-md border border-input bg-background px-3 py-2 text-sm ring-offset-background file:border-0 file:bg-transparent file:text-sm file:font-medium placeholder:text-muted-foreground focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-ring focus-visible:ring-offset-2 disabled:cursor-not-allowed disabled:opacity-50';
const DEFAULT_CLASS_FORMATTED_POSITIVE = 'text-green-600';
const DEFAULT_CLASS_FORMATTED_NEGATIVE = 'text-red-600';
const DEFAULT_CLASS_FORMATTED_ZERO = 'currencyInput__formatted--zero';

type Callback = (value: number) => any;

// export let value: number = DEFAULT_VALUE;
export let value: $$Props['value'] = DEFAULT_VALUE;

export let locale: string = DEFAULT_LOCALE;
export let currency: string = DEFAULT_CURRENCY;
export let name: string = DEFAULT_NAME;
export let required: boolean = false;
export let disabled: boolean = false;
export let placeholder: string | number | null = DEFAULT_VALUE;
export let autocomplete: string | null | undefined = undefined;
export let isNegativeAllowed: boolean = true;
export let isZeroNullish: boolean = false;
export let fractionDigits: number = DEFAULT_FRACTION_DIGITS;
export let inputClasses: InputClasses | null = null;
export let onValueChange: Callback = () => {};

// Formats value as: e.g. $1,523.00 | -$1,523.00
const formatCurrency = (
	value: number,
	maximumFractionDigits?: number,
	minimumFractionDigits?: number
) => {
	return new Intl.NumberFormat(locale, {
		currency: currency,
		style: 'currency',
		maximumFractionDigits: maximumFractionDigits || 0,
		minimumFractionDigits: minimumFractionDigits || 0
	}).format(value);
};

// Checks if the key pressed is allowed
const handleKeyDown = (event: KeyboardEvent) => {
	const isDeletion = event.key === 'Backspace' || event.key === 'Delete';
	const isModifier = event.metaKey || event.altKey || event.ctrlKey;
	const isArrowKey = event.key === 'ArrowLeft' || event.key === 'ArrowRight';
	const isTab = event.key === 'Tab';
	const isInvalidCharacter = !/^\d|,|\.|-$/g.test(event.key); // Keys that are not a digit, comma, period or minus sign

	// If there is already a decimal point, don't allow more than one
	const isPunctuationDuplicated = () => {
		if (event.key !== ',' && event.key !== '.') return false; // Is `false` because it's not a punctuation key
		if (isDecimalComma) return formattedValue.split(',').length >= 2;
		if (!isDecimalComma) return formattedValue.split('.').length >= 2;
		return false;
	};

	if (
		isPunctuationDuplicated() ||
		(!isDeletion && !isModifier && !isArrowKey && isInvalidCharacter && !isTab)
	)
		event.preventDefault();
};

// Formats the value when the input loses focus and sets the correct number of
// fraction digits when the value is zero
const handleOnBlur = () => setFormattedValue();

let dom: Document;
let inputElement: HTMLInputElement;

onMount(() => {
	// Set the document object as a variable so we know the page has mounted
	dom = document;

	// Set the correct fraction digits when the value is zero on initial load
	setFormattedValue();
});

let inputTarget: EventTarget | null;
const currencyDecimal = new Intl.NumberFormat(locale).format(1.1).charAt(1); // '.' or ','
const isDecimalComma = currencyDecimal === ',';
const currencySymbol = formatCurrency(0, 0)
	.replace('0', '') // e.g. '$0' > '$'
	.replace(/\u00A0/, ''); // e.g '0 €' > '€'

// Updates `value` by stripping away the currency formatting
const setUnformattedValue = (event?: KeyboardEvent) => {
	if (event) {
		// Don't format if the user is typing a `currencyDecimal` point
		if (event.key === currencyDecimal) return;

		// Pressing `.` when the decimal point is `,` gets replaced with `,`
		if (isDecimalComma && event.key === '.')
			formattedValue = formattedValue.replace(/\.([^.]*)$/, currencyDecimal + '$1'); // Only replace the last occurence

		// Pressing `,` when the decimal point is `.` gets replaced with `.`
		if (!isDecimalComma && event.key === ',')
			formattedValue = formattedValue.replace(/\,([^,]*)$/, currencyDecimal + '$1'); // Only replace the last occurence

		// Don't format if `formattedValue` is ['$', '-$', "-"]
		const ignoreSymbols = [currencySymbol, `-${currencySymbol}`, '-'];
		const strippedUnformattedValue = formattedValue.replace(' ', '');
		if (ignoreSymbols.includes(strippedUnformattedValue)) return;

		// Set the starting caret positions
		inputTarget = event.target;

		// Reverse the value when minus is pressed
		if (isNegativeAllowed && event.key === '-') value = value * -1;
	}

	// Remove all characters that arent: numbers, commas, periods (or minus signs if `isNegativeAllowed`)
	let unformattedValue = isNegativeAllowed
		? formattedValue.replace(/[^0-9,.-]/g, '')
		: formattedValue.replace(/[^0-9,.]/g, '');

	// Finally set the value
	if (Number.isNaN(parseFloat(unformattedValue))) {
		value = 0;
	} else {
		// The order of the following operations is *critical*
		unformattedValue = unformattedValue.replace(isDecimalComma ? /\./g : /\,/g, ''); // Remove all group symbols
		if (isDecimalComma) unformattedValue = unformattedValue.replace(',', '.'); // If the decimal point is a comma, replace it with a period

		// If the zero-key has been pressed
		// and if the current `value` is the same as the `value` before the key-press
		// formatting may need to be done (Issue #30)
		const previousValue = value;
		value = parseFloat(unformattedValue);

		if (event && previousValue === value) {
			// Do the formatting if the number of digits after the decimal point exceeds `fractionDigits`
			if (
				unformattedValue.includes('.') &&
				unformattedValue.split('.')[1].length > fractionDigits
			) {
				setFormattedValue();
			}
		}
	}
};

const setFormattedValue = () => {
	// Do nothing because the page hasn't mounted yet
	if (!dom) return;

	// Previous caret position
	const startCaretPosition = inputElement?.selectionStart || 0;
	const previousFormattedValueLength = formattedValue.length;

	// Apply formatting to input
	formattedValue =
		isZero && !isZeroNullish
			? ''
			: formatCurrency(
					value,
					fractionDigits,
					dom.activeElement === inputElement ? 0 : fractionDigits
				);

	// Update `value` after formatting
	setUnformattedValue();

	let retries = 0;
	while (previousFormattedValueLength === formattedValue.length && retries < 10) retries++;

	if (previousFormattedValueLength !== formattedValue.length) {
		const endCaretPosition =
			startCaretPosition + formattedValue.length - previousFormattedValueLength;
		inputElement?.setSelectionRange(endCaretPosition, endCaretPosition);
	}

	// Run callback function when `value` changes
	onValueChange(value);
};

const handlePlaceholder = (placeholder: string | number | null) => {
	if (typeof placeholder === 'number')
		return formatCurrency(placeholder, fractionDigits, fractionDigits);
	if (placeholder === null) return '';
	return placeholder;
};

let formattedValue = '';
$: isNegative = value < 0;
$: isPositive = value > 0;
$: isZero = !isNegative && !isPositive;
$: value, setFormattedValue();
</script>

<input
	class={inputClasses?.unformatted ?? DEFAULT_CLASS_UNFORMATTED}
	type="hidden"
	name={name}
	disabled={disabled}
	bind:value={value}
/>
<input
	class={cn(
			inputClasses?.formatted ?? DEFAULT_CLASS_FORMATTED,
			isNegativeAllowed && !isZero && !isNegative
			? inputClasses?.formattedPositive ?? DEFAULT_CLASS_FORMATTED_POSITIVE
			: '',
			isZero ? inputClasses?.formattedZero ?? DEFAULT_CLASS_FORMATTED_ZERO : '',
			isNegativeAllowed && isNegative
			? inputClasses?.formattedNegative ?? DEFAULT_CLASS_FORMATTED_NEGATIVE
			: '',
			className
		)}
	readonly={readonly}
	type="text"
	inputmode={fractionDigits > 0 ? 'decimal' : 'numeric'}
	name={`formatted-${name}`}
	required={required && !isZero}
	placeholder={handlePlaceholder(placeholder)}
	autocomplete={autocomplete}
	disabled={disabled}
	bind:this={inputElement}
	bind:value={formattedValue}
	on:keydown={handleKeyDown}
	on:keyup={setUnformattedValue}
	on:blur={handleOnBlur}
	on:change
	on:click
	on:focus
	on:focusin
	on:focusout
	on:keypress
	on:mouseover
	on:mouseenter
	on:mouseleave
	on:paste
	on:input
	on:wheel
	{...$$restProps}
/>
