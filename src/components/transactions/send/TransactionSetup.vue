<template>
	<div>
		<div class="identifier">
			<identicon :value="recipientAddress" />
		</div>
		<div class="control">
			<label>{{ translate('sendSiacoinsModal.recipientAddress') }}</label>
			<input type="text" v-model="recipientAddress" :placeholder="translate('sendSiacoinsModal.txtRecipientPlaceholder')" />
		</div>
		<label>{{ translate('amount') }}</label>
		<div class="currency-control">
			<input ref="txtSiacoin" type="text" value="0 SC" @input="onChangeSiacoin" @blur="onFormatValues" />
			<label>{{ translate('currency.sc') }}</label>
			<input ref="txtCurrency" type="text" value="$0.00" @input="onChangeCurrency" @blur="onFormatValues" />
			<label>{{ translate(`currency.${currency}`) }}</label>
		</div>
		<div class="extras-info">
			<div>{{ translate('transactionFee') }}</div>
			<div class="text-right" v-html="transactionFeeSC" />
			<div class="text-right" v-html="transactionFeeCurrency" />
			<div>{{ translate('sendSiacoinsModal.spendableBalance') }}</div>
			<div class="text-right" v-html="remainingBalanceSC" />
			<div class="text-right" v-html="remainingBalanceCurrency" />
		</div>
		<div class="transaction-error text-center text-warning">
			<transition name="fade" mode="out-in" appear>
				<div v-if="transactionError" :key="transactionError">{{ transactionError }}</div>
				<div v-else class="error-hidden">hidden</div>
			</transition>
		</div>
		<div class="buttons">
			<button class="btn btn-success btn-inline" :disabled="transactionError || sending" @click="onSendTxn">{{ translate('send') }}</button>
		</div>
	</div>
</template>

<script>
import BigNumber from 'bignumber.js';
import { mapState } from 'vuex';
import { verifyAddress } from '@/utils';
import { parseCurrencyString, parseSiacoinString } from '@/utils/parse';
import { formatPriceString } from '@/utils/format';
import { getWalletAddresses } from '@/store/db';

import Identicon from '@/components/Identicon';

export default {
	components: {
		Identicon
	},
	props: {
		address: String,
		wallet: Object
	},
	computed: {
		...mapState(['currency', 'currencies', 'networkFees']),
		walletBalance() {
			return this.wallet.unconfirmedSiacoinBalance();
		},
		changeAddress() {
			let addr = this.ownedAddresses.find(a => a.usage_type !== 'sent');

			if (!addr)
				addr = this.ownedAddresses[this.ownedAddresses.length - 1];

			return addr;
		},
		unspent() {
			const outputs = this.wallet && Array.isArray(this.wallet.unspent_siacoin_outputs) ? this.wallet.unspent_siacoin_outputs : [],
				spent = this.wallet && Array.isArray(this.wallet.spent_siacoin_outputs) ? this.wallet.spent_siacoin_outputs : [],
				unspent = outputs.filter(o => spent.indexOf(o.output_id) === -1);

			if (!Array.isArray(unspent) || unspent.length === 0)
				return [];

			unspent.sort((a, b) => {
				a = new BigNumber(a.value);
				b = new BigNumber(b.value);

				if (a.gt(b))
					return 1;

				if (a.lt(b))
					return -1;

				return 0;
			});

			return unspent;
		},
		transactionFeeSC() {
			const siacoins = formatPriceString(this.fees, 2);

			return `${siacoins.value} <span class="currency-display">${this.translate('currency.sc')}</span>`;
		},
		transactionFeeCurrency() {
			const currency = formatPriceString(this.fees, 2, this.currency, this.currencies[this.currency]);

			return `${currency.value} <span class="currency-display">${this.translate(`currency.${currency.label}`)}</span>`;
		},
		calculatedAmount() {
			let amount = this.sendAmount;

			if (this.ownedAddresses.indexOf(this.recipientAddress) !== -1)
				amount = new BigNumber(0);

			return amount;
		},
		remainingBalanceSC() {
			const rem = this.walletBalance.minus(this.calculatedAmount).minus(this.fees),
				siacoins = formatPriceString(rem, 2);

			return `${siacoins.value} <span class="currency-display">${this.translate('currency.sc')}</span>`;
		},
		remainingBalanceCurrency() {
			const rem = this.walletBalance.minus(this.calculatedAmount).minus(this.fees),
				currency = formatPriceString(rem, 2, this.currency, this.currencies[this.currency]);

			return `${currency.value} <span class="currency-display">${this.translate(`currency.${currency.label}`)}</span>`;
		},
		transactionError() {
			if (this.sendAmount.lte(0))
				return this.translate('sendSiacoinsModal.errorGreaterThan0');

			if (this.sendAmount.plus(this.fees).gt(this.walletBalance))
				return this.translate('sendSiacoinsModal.errorNotEnough');

			if (this.sendAmount.lt(this.fees))
				return this.translate('sendSiacoinsModal.errorHighFee');

			if (!verifyAddress(this.recipientAddress))
				return this.translate('sendSiacoinsModal.errorBadRecipient');

			return null;
		},
		minInputs() {
			return this.fundTransaction(this.sendAmount).inputs.length;
		},
		minOutputs() {
			return 3;
		},
		txnSize() {
			return 100 + (this.minInputs * 313) + (this.minOutputs * 50);
		},
		apiFee() {
			return new BigNumber(this.networkFees.api.fee).times(this.txnSize);
		},
		siaFee() {
			return new BigNumber(this.networkFees.minimum).plus(this.networkFees.maximum).div(2).times(this.txnSize);
		},
		fees() {
			return this.apiFee.plus(this.siaFee);
		}
	},
	data() {
		return {
			recipientAddress: '',
			sendAmount: new BigNumber(0),
			sending: false,
			ownedAddresses: []
		};
	},
	async mounted() {
		try {
			if (typeof this.address === 'string' && this.address.length > 0)
				this.recipientAddress = this.address;

			this.onFormatValues();
			await this.loadAddresses();
		} catch (ex) {
			console.error('TransactionSetupMounted', ex);
			this.pushNotification({
				severity: 'danger',
				message: ex.message
			});
		}
	},
	methods: {
		async loadAddresses() {
			this.ownedAddresses = await getWalletAddresses(this.wallet.id);

			if (this.ownedAddresses.length === 0)
				throw new Error('no addresses');
		},
		ownsAddress(address) {
			return this.ownedAddresses.findIndex(a => a.address === address && a.unlock_conditions) !== -1;
		},
		fundTransaction(amount) {
			const inputs = [];
			let added = new BigNumber(0);

			for (let i = 0; i < this.unspent.length; i++) {
				const output = this.unspent[i],
					addr = this.ownedAddresses.find(a => output.unlock_hash === a.address && a.unlock_conditions);

				if (!addr)
					continue;

				added = added.plus(output.value);
				inputs.push({
					...output,
					...addr,
					owned: true
				});

				if (added.gte(amount))
					break;
			}

			return {
				inputs,
				added
			};
		},
		buildTransaction() {
			const { inputs, added } = this.fundTransaction(this.sendAmount.plus(this.fees)),
				txn = {
					miner_fees: [this.siaFee.toString(10)],
					siacoin_inputs: inputs,
					siacoin_outputs: []
				},
				feeAddress = this.networkFees.api.address,
				change = added.minus(this.fees).minus(this.sendAmount);

			if (added.lt(this.sendAmount.plus(this.fees)))
				throw new Error('not enough funds to create transaction');

			txn.siacoin_outputs.push(
				{
					unlock_hash: this.recipientAddress,
					value: this.sendAmount.toString(10),
					tag: 'Recipient',
					owned: this.ownsAddress(this.recipientAddress)
				},
				{
					unlock_hash: feeAddress,
					value: this.apiFee.toString(10),
					tag: 'API Fee',
					owned: false
				}
			);

			if (change.gt(0)) {
				if (!this.changeAddress || !this.changeAddress.address || !verifyAddress(this.changeAddress.address))
					throw new Error('unable to send transaction. no change address');

				txn.siacoin_outputs.push({
					unlock_hash: this.changeAddress.address,
					value: change.toString(10),
					tag: 'Change',
					owned: this.ownsAddress(this.changeAddress.address)
				});
			}

			return txn;
		},
		formatCurrencyString(value) {
			return formatPriceString(value, 2, this.currency, this.currencies[this.currency]).value;
		},
		async onSendTxn() {
			if (this.sending)
				return;

			this.sending = true;

			try {
				this.$emit('built', this.buildTransaction());
			} catch (ex) {
				console.error('onSendTxn', ex);
				this.pushNotification({
					severity: 'danger',
					message: ex.message
				});
			} finally {
				this.sending = false;
			}
		},
		onFormatValues() {
			try {
				const siacoins = formatPriceString(this.sendAmount, 2);

				this.$refs.txtCurrency.value = this.formatCurrencyString(this.sendAmount);
				this.$refs.txtSiacoin.value = siacoins.value;
			} catch (ex) {
				console.error('onFormatValues', ex);
				this.pushNotification({
					severity: 'danger',
					message: ex.message
				});
			}
		},
		onChangeSiacoin() {
			try {
				const value = this.$refs.txtSiacoin.value,
					parsed = parseSiacoinString(value);

				this.sendAmount = parsed;
				this.$refs.txtCurrency.value = this.formatCurrencyString(parsed);
			} catch (ex) {
				console.error('onChangeSiacoin', ex);
				this.pushNotification({
					severity: 'danger',
					message: ex.message
				});
			}
		},
		onChangeCurrency() {
			try {
				const value = this.$refs.txtCurrency.value,
					parsed = parseCurrencyString(value, this.currencies[this.currency]),
					siacoins = formatPriceString(parsed, 2);

				this.sendAmount = parsed;
				this.$refs.txtSiacoin.value = siacoins.value;
			} catch (ex) {
				console.error('onChangeCurrency', ex);
				this.pushNotification({
					severity: 'danger',
					message: ex.message
				});
			}
		}
	},
	watch: {
		address() {
			if (typeof this.address === 'string' && this.address.length > 0)
				this.recipientAddress = this.address;
		}
	}
};
</script>

<style lang="stylus" scoped>
.currency-control {
	display: grid;
	grid-template-columns: minmax(0, 1fr) auto;
	margin-bottom: 15px;

	input, label {
		height: 36px;
		line-height: 36px;
		padding: 0 5px;
	}

	label {
		display: inline-block;
		color: rgba(255, 255, 255, 0.54);
		text-transform: uppercase;
		margin: 0;
	}

	input {
		display: block;
		width: 100%;
		font-size: 1.2rem;
		background: transparent;
		border: 1px solid dark-gray;
		color: rgba(255, 255, 255, 0.84);
		outline: none;
		text-align: right;

		&:first-of-type {
			border-top-left-radius: 4px;
			border-top-right-radius: 4px;
		}

		&:last-of-type {
			border-bottom-left-radius: 4px;
			border-bottom-right-radius: 4px;
			border-top: none;
		}
	}
}

.extras-info {
	display: grid;
	grid-template-columns: minmax(0, 1fr) repeat(2, auto);
	grid-gap: 10px;
	margin-bottom: 15px;
}

.transaction-error {
	margin-bottom: 15px;

	.error-hidden {
		opacity: 0;
	}
}

.identifier {
	width: 100px;
	margin: auto auto 30px;

	svg {
		width: 100%;
		height: 100%;
		border-radius: 4px;
	}
}
</style>