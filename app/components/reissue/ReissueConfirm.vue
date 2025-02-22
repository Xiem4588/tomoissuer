<template>
    <div class="container container-small flex-content-center">
        <div class="confirm-table tomo-body-fullw">
            <div class="info-header">
                <h2 class="tmp-title-large m-0">{{ token.name }} Token Reissue</h2>
                <p class="text-center mt-5"><i class="tomoissuer-icon-burn"/></p>
                <div class="text-center mb-5">
                    <strong>+ {{ formatNumber(reissueAmount) }} {{ token.symbol }}</strong>
                </div>
            </div>
            <div class="tmp-table-two grid-two">
                <table>
                    <tr>
                        <td>Owner balance after reissuing</td>
                        <td>
                            {{ formatNumber(newOwnerbalance) }}
                        </td>
                    </tr>
                    <tr>
                        <td>Total supply after reissuing</td>
                        <td>
                            {{ formatNumber(newTotalSupply) }}
                        </td>
                    </tr>
                    <tr>
                        <td>Transaction fee</td>
                        <td>{{ formatNumber(txFee) }} {{ token.symbol }}</td>
                    </tr>
                </table>
            </div>
            <div class="qr-code-donate text-center d-none">
                <p>
                    <b-img
                        src="/app/assets/images/img-qrcode-tomowallet.png"
                        alt="img-qrcode-tomowallet.png"/>
                </p>
                <p>
                    <b>Scan QR code below using TomoWallet to donate</b>
                </p>
            </div>
            <div class="btn-box">
                <b-button
                    :to="'/reissueToken/' + address"
                    class="tmp-btn-boder-blue btn-min">
                    Back
                </b-button>
                <b-button
                    class="tmp-btn-blue"
                    @click="reissue">
                    Reissue
                </b-button>
            </div>
            <b-modal
                id="modal-deposit"
                ref="poolingFeeModal"
                size="md"
                hide-header
                hide-footer
                centered
                no-close-on-esc
                no-close-on-backdrop>
                <div class="tomo-modal-default icon-violet">
                    <div class="msg-txt">
                        <i class="tomoissuer-icon-checkmark-outline"/>
                        <h4>Successful</h4>
                        <p>You’ve just reissued {{ formatNumber(reissueAmount) }} {{ token.symbol }}</p>
                        <p>
                            Transaction hash:
                            <a
                                :href="config.tomoscanUrl + '/txs/' +
                                transactionHash.toLowerCase()"
                                :title="transactionHash"
                                target="_blank">
                                {{ truncate(transactionHash || '123456', 26) }}
                            </a>
                        </p>
                    </div>
                    <div class="btn-box">
                        <b-button
                            :to="'/token/' + address"
                            class="tmp-btn-violet">
                            Token detail
                        </b-button>
                    </div>
                </div>
            </b-modal>
        </div>
        <div
            :class="(loading ? 'tomo-loading' : '')"/>
    </div>
</template>

<script>
import store from 'store'
import axios from 'axios'
import BigNumber from 'bignumber.js'
export default {
    name: 'ReissueConfirm',
    data () {
        return {
            address: this.$route.params.address.toLowerCase(),
            account: '',
            transactionHash: '',
            reissueAmount: this.$route.params.reissueAmount,
            token: {},
            config: {},
            loading: false,
            gasPrice: '',
            txFee: '',
            ownerBalance: ''
        }
    },
    computed: {
        newOwnerbalance: function () {
            return (new BigNumber(this.ownerBalance).plus(this.reissueAmount))
        },
        newTotalSupply: function () {
            return (new BigNumber(this.token.totalSupplyNumber).plus(this.reissueAmount))
        }
    },
    async updated () {},
    destroyed () { },
    created: async function () {
        this.account = store.get('address') ||
            this.$store.state.address || await this.getAccount()
        if (!this.account) {
            this.$router.push({ path: '/login' })
        }

        this.config = store.get('configIssuer') || await this.appConfig()

        this.web3.eth.getGasPrice().then(result => {
            this.gasPrice = result
        }).catch(error => {
            console.log(error)
            this.$toasted.show(error, { type: 'error' })
        })
        await this.getTokenDetail()
        this.getTransactionFee()
        this.getOwnerBalance()
    },
    methods: {
        async getTokenDetail () {
            const self = this
            const { data } = await axios.get(`/api/account/${self.address}`)
            const token = data.token
            self.token = token || {}
            self.contractCreation = data.contractCreation.toLowerCase()
        },
        getOwnerBalance () {
            const web3 = this.web3
            if (this.contractCreation && web3) {
                // 0x70a08231 is balanceOf(address) function code
                let data = '0x70a08231' +
                    '000000000000000000000000' +
                    this.contractCreation.substr(2) // chop off the 0x
                web3.eth.call({ to: this.address, data: data }).then(result => {
                    let balance = new BigNumber(web3.utils.hexToNumberString(result))
                    this.ownerBalance = balance.div(10 ** this.token.decimals)
                }).catch(error => {
                    console.log(error)
                    this.$toatsed.show(error, { type: 'error' })
                })
            }
        },
        getTransactionFee () {
            const web3 = this.web3
            if (this.account && web3) {
                // 0x24ec7590 is minFee function code
                let data = '0x24ec7590' +
                    '00000000000000000000000000000000000000000000000000000000'
                web3.eth.call({ to: this.address, data: data }).then(result => {
                    let balance = new BigNumber(web3.utils.hexToNumberString(result))
                    this.txFee = balance.div(10 ** this.token.decimals).toNumber()
                }).catch(error => {
                    console.log(error)
                    this.$toatsed.show(error, { type: 'error' })
                })
            }
        },
        async reissue () {
            try {
                if (this.reissueAmount && this.contractCreation === this.account.toLowerCase()) {
                    this.loading = true
                    const chainConfig = this.config.blockchain
                    const txParams = {
                        from: this.account,
                        gasPrice: this.web3.utils.toHex(this.gasPrice),
                        gas: this.web3.utils.toHex(chainConfig.gas),
                        gasLimit: this.web3.utils.toHex(chainConfig.gas)
                    }

                    const { data } = await axios.get('/api/token/getABI?type=mintable')
                    if (data.abi) {
                        const contract = new this.web3.eth.Contract(
                            data.abi,
                            this.address
                        )
                        // const contract = this.TRC21Issuer
                        const provider = this.NetworkProvider
                        if (provider === 'ledger' || provider === 'trezor') {
                            let data = await contract.methods.mint(
                                this.contractCreation,
                                (new BigNumber(this.reissueAmount).multipliedBy(10 ** this.token.decimals)).toString(10)
                            ).encodeABI()

                            const dataTx = {
                                data,
                                to: this.address
                            }
                            // bypass hardware wallet to sign tx
                            txParams.value = this.web3.utils.toHex(0)
                            let nonce = await this.web3.eth.getTransactionCount(this.account)
                            Object.assign(
                                dataTx,
                                dataTx,
                                txParams,
                                {
                                    nonce: this.web3.utils.toHex(nonce)
                                }
                            )
                            let signature = await this.signTransaction(dataTx)
                            delete dataTx.value
                            const txHash = await this.sendSignedTransaction(dataTx, signature)
                            if (txHash) {
                                this.transactionHash = txHash
                                let check = true
                                while (check) {
                                    const receipt = await this.web3.eth.getTransactionReceipt(txHash)
                                    if (receipt) {
                                        self.loading = false
                                        check = false
                                        this.$refs.poolingFeeModal.show()
                                    }
                                }
                            }
                        } else {
                            contract.methods.mint(
                                this.contractCreation,
                                (new BigNumber(this.reissueAmount).multipliedBy(10 ** this.token.decimals)).toString(10)
                            ).send(txParams)
                                .on('transactionHash', async (txHash) => {
                                    this.transactionHash = txHash
                                    let check = true
                                    while (check) {
                                        const receipt = await this.web3.eth.getTransactionReceipt(txHash)
                                        if (receipt) {
                                            self.loading = false
                                            check = false
                                            this.$refs.poolingFeeModal.show()
                                        }
                                    }
                                })
                                .catch(error => {
                                    console.log(error)
                                    this.loading = false
                                    this.$toasted.show(error, { type: 'error' })
                                })
                        }
                    }
                }
            } catch (error) {
                console.log(error)
                this.loading = false
                this.$toasted.show(error, { type: 'error' })
            }
        }
    }
}
</script>
