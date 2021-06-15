<template>
  <div class="app-container">
    <el-row>
      <el-col :span="24">
        <el-alert
          type="warning"
          show-icon
          :closable="false"
        >
          <template slot="title">
            <div class="iconSize">风险提示：</div>
            <div class="iconSize">1. 本网站代码已开源，地址：https://github.com/jiangydev/starcoin-tools；</div>
            <div class="iconSize">2. 本页面功能使用前，请确保打开的不是钓鱼网站；</div>
            <div class="iconSize">3. 若有任何疑问, 请在社区群内联系开发者【安_change】；</div>
            <div class="iconSize">STC 捐赠地址: 0x8b79fdf7bd004b72ea4bd83289429455 / stc1p3dulmaaaqp9h96jtmqegjs552hhj0qt5jw0x2qen6p42xma68exgk70a777sqjmjaf9asv5fg22929qzpup</div>
          </template>
        </el-alert>
      </el-col>
    </el-row>
    <el-row>
      <el-col :span="24">
        <el-card shadow="always">
          <el-divider>Starcoin Move 合约标准库 v5 版升级</el-divider>
          <el-link type="primary" href="https://news.starcoin.org/zh/2021/starcoin_stdlib_upgrade_v5/">点此查看官方详细通告</el-link><br><br>
          Starcoin 主网上线以来第一次升级，主要包含以下特性：<br><br>
          1. 从国库提款的时候增加额度限制，最大数额不能超过投票通过阈值（当前流通量的 4%）。<br>
          2. 实现了新的链上认证策略，简化初始化链上账号的复杂度。
          <el-divider>投票状态</el-divider>
          <el-steps :active="proposalInfo.state - 1" finish-status="success">
            <el-step title="⭐等待公示中" />
            <el-step title="🔥正在投票中" />
            <el-step title="😭提案被拒绝" />
            <el-step title="✌投票通过" />
            <el-step title="🚓执行排队中" />
            <el-step title="💣可执行/待触发" />
            <el-step title="🚀已经执行" />
          </el-steps>
          <el-divider>投票详情</el-divider>
          开始时间：{{ proposalInfo.startTime }}
          <el-divider direction="vertical" />
          结束时间：{{ proposalInfo.endTime }}<br><br>
          赞成票数：{{ proposalInfo.agreeSTC / Math.pow(10, 9) }} STC
          <el-divider direction="vertical" />
          反对票数：{{ proposalInfo.disagreeSTC / Math.pow(10, 9) }} STC
        </el-card>
      </el-col>
    </el-row>
    <el-row>
      <el-col :span="24">
        <el-card shadow="always">
          <el-form ref="ruleForm" :model="ruleForm" :rules="rules" label-width="130px" class="vote-ruleForm">
            <el-form-item label="投票账户私钥" prop="privateKey">
              <el-row align="middle" type="flex">
                <el-col :span="16">
                  <el-input v-model="ruleForm.privateKey" :disabled="ruleForm.keyDisable" autocomplete="off" />
                </el-col>
                <el-col :span="4">
                  <el-switch
                    v-model="ruleForm.keyLock"
                    style="display: block"
                    active-color="#13ce66"
                    inactive-color="#ff4949"
                    active-text="锁定"
                    inactive-text=" "
                    @change="changeKeyLock"
                  />
                </el-col>
                <el-col :span="4">
                  <el-button
                    type="primary"
                    icon="el-icon-refresh"
                    :disabled="!ruleForm.keyDisable"
                    :loading="voteLoading"
                    @click="getAccountInfo()"
                  >刷新账户余额</el-button>
                </el-col>
              </el-row>
            </el-form-item>
            <el-form-item label="当前账户地址">
              <span>{{ accountInfo.address }}</span>
            </el-form-item>
            <el-form-item label="当前账户余额">
              <span>{{ accountInfo.balance / Math.pow(10, 9) }} STC</span>
            </el-form-item>
            <el-form-item label="当前账户已投票数">
              <span>{{ accountInfo.voteCount / Math.pow(10, 9) }} STC</span>
            </el-form-item>
            <el-divider>质押投票</el-divider>
            <el-form-item label="是否同意提案" prop="agree">
              <el-radio-group v-model="ruleForm.agree">
                <el-radio label="true">同意</el-radio>
                <el-radio label="false">反对</el-radio>
              </el-radio-group>
            </el-form-item>
            <el-form-item label="投票数量(STC)" prop="amount">
              <el-input-number v-model="ruleForm.amount" :precision="4" :step="0.1" :min="0.0001" :max="999999999" />
            </el-form-item>
            <el-form-item>
              <el-button type="primary" :loading="voteLoading" :disabled="!ruleForm.keyDisable" @click="submitForm('ruleForm')">立即投票</el-button>
              <el-button @click="resetForm('ruleForm')">重置</el-button>
            </el-form-item>
            <el-divider>质押取回</el-divider>
            <el-form-item>
              <el-button type="primary" :loading="voteLoading" :disabled="!ruleForm.keyDisable" @click="unstakeVote('ruleForm')">取回抵押</el-button>
            </el-form-item>
          </el-form>
        </el-card>
      </el-col>
    </el-row>
  </div>
</template>

<script>
import * as starcoin from '@starcoin/starcoin'
import { arrayify, hexlify } from '@ethersproject/bytes'
import { BigNumber } from 'bignumber.js'
import moment from 'moment'

export default {
  data() {
    return {
      provider: null,
      starcoinNetwork: {},
      accountInfo: {
        address: '',
        publicKey: '',
        balance: 0,
        voteCount: 0
      },
      proposalInfo: {
        state: 0,
        proposalId: null,
        startTime: null,
        endTime: null,
        agreeSTC: null,
        disagreeSTC: null
      },
      ruleForm: {
        privateKey: '',
        keyDisable: false,
        keyLock: false,
        agree: 'true',
        amount: ''
      },
      rules: {
        privateKey: [
          { required: true, message: '请输入私钥', trigger: 'blur' },
          { min: 66, max: 66, message: '应该输入 0x 前缀, 且总长度为 66 的私钥', trigger: 'blur' }
        ],
        agree: [
          { required: true, message: '请选择是否同意', trigger: 'change' }
        ],
        amount: [
          { required: true, message: '请输入投票数量', trigger: 'blur' }
        ]
      },
      voteLoading: false
    }
  },
  created() {
    this.initStarcoin().then(() => {
      this.initVoteInfo()
    })
  },
  methods: {
    // 初始化 provider 和检测网络
    async initStarcoin() {
      this.provider = new starcoin.providers.JsonRpcProvider('https://main-seed.starcoin.org')
      const networkRsp = await this.provider.detectNetwork()
      // console.log('当前网络 -> ', networkRsp)
      this.starcoinNetwork = {
        chainId: networkRsp.chainId
      }
    },
    async initVoteInfo() {
      // 获取投票状态
      const proposalStateRsp = await this.provider.call({
        function_id: '0x1::Dao::proposal_state',
        type_args: ['0x1::STC::STC', '0x1::UpgradeModuleDaoProposal::UpgradeModuleV2'],
        args: ['0xb2aa52f94db4516c5beecef363af850a', '0']
      })
      // console.log('提案状态 -> ', proposalStateRsp)
      this.proposalInfo.state = proposalStateRsp[0]
      // 获取投票详情
      const proposalInfoRsp = await this.provider.call({
        function_id: '0x1::Dao::proposal_info',
        type_args: ['0x1::STC::STC', '0x1::UpgradeModuleDaoProposal::UpgradeModuleV2'],
        args: ['0xb2aa52f94db4516c5beecef363af850a']
      })
      this.proposalInfo.proposalId = proposalInfoRsp[0]
      const startTime = proposalInfoRsp[1]
      this.proposalInfo.startTime = moment(startTime).format('YYYY-MM-DD HH:mm:ss')
      const endTime = proposalInfoRsp[2]
      this.proposalInfo.endTime = moment(endTime).format('YYYY-MM-DD HH:mm:ss')
      this.proposalInfo.agreeSTC = proposalInfoRsp[3]
      this.proposalInfo.disagreeSTC = proposalInfoRsp[4]
      // console.log('提案详情 -> ', proposalInfoRsp)
    },
    // 锁定/解锁私钥
    async changeKeyLock() {
      if (this.ruleForm.keyLock === true) {
        if (this.ruleForm.privateKey === '') {
          this.ruleForm.keyLock = false
          this.$message({
            type: 'error',
            message: '请先输入私钥!'
          })
          return
        }
        this.ruleForm.keyDisable = true
        // 发送者公钥
        const senderPublicKeyHex = await starcoin.encoding.privateKeyToPublicKey(this.ruleForm.privateKey)
        this.accountInfo.publicKey = senderPublicKeyHex
        // 发送者地址
        const senderAddressHex = starcoin.encoding.publicKeyToAddress(senderPublicKeyHex)
        this.accountInfo.address = senderAddressHex
        await this.getAccountInfo()
      } else {
        this.ruleForm.keyDisable = false
      }
    },
    // 获取账户信息
    async getAccountInfo() {
      this.voteLoading = true
      try {
        const balance = await this.provider.getBalance(this.accountInfo.address)
        this.accountInfo.balance = balance
        // 获取账户已投票数
        const voteResource = await this.provider.getResource(this.accountInfo.address, '0x1::Dao::Vote<0x1::STC::STC>')
        if (voteResource !== undefined) {
          this.accountInfo.voteCount = voteResource.stake.value
        } else {
          this.accountInfo.voteCount = 0
        }
      } finally {
        this.voteLoading = false
      }
    },
    // 投票
    submitForm(formName) {
      this.$refs[formName].validate((valid) => {
        if (valid) {
          this.voteLoading = true
          if (this.proposalInfo.state !== 2) {
            throw new Error('当前提案状态无法参与投票')
          }
          let agree = true
          if (this.ruleForm.agree === 'false') {
            agree = false
          }
          const amount = this.ruleForm.amount * Math.pow(10, 9)
          this.onSubmit(
            this.ruleForm.privateKey,
            amount,
            agree
          )
        } else {
          console.log('error submit!!')
          return false
        }
      })
    },
    unstakeVote(formName) {
      this.$refs[formName].validate((valid) => {
        if (valid) {
          this.voteLoading = true
          if (this.proposalInfo.state <= 2) {
            throw new Error('当前提案状态无法取回质押')
          }
          this.onUnstakeVoteSubmit(
            this.ruleForm.privateKey
          )
        } else {
          console.log('error submit!!')
          return false
        }
      })
    },
    resetForm(formName) {
      this.$refs[formName].resetFields()
      this.ruleForm.keyLock = false
      this.ruleForm.keyDisable = false
    },
    async onUnstakeVoteSubmit(senderPrivateKeyHex) {
      this.voteLoading = true
      try {
        const provider = this.provider
        if (provider === null) {
          throw new Error('Starcoin 网络未准备好，请刷新页面重试')
        }
        const chainId = this.starcoinNetwork.chainId
        const senderAddressHex = this.accountInfo.address
        const senderPublicKeyHex = this.accountInfo.publicKey
        // senderAddressHex = '0x8b79fdf7bd004b72ea4bd83289429455'
        // 获取账号交易数
        const senderSequenceNumber = await provider.getSequenceNumber(senderAddressHex)

        // 提案发起者地址
        const receiver = '0xb2aa52f94db4516c5beecef363af850a'
        let receiverAddressHex = ''
        if (receiver.slice(0, 3) === 'stc') {
          const receiptIdentifier = starcoin.starcoin_types.ReceiptIdentifier.decode(receiver)
          receiverAddressHex = starcoin.encoding.addressFromSCS(receiptIdentifier.accountAddress)
        } else {
          receiverAddressHex = receiver
        }

        // 构建交易请求体
        const txnRequest = {
          chain_id: chainId,
          gas_unit_price: 1,
          sender: senderAddressHex,
          sender_public_key: senderPublicKeyHex,
          sequence_number: senderSequenceNumber,
          max_gas_amount: 10000000,
          script: {
            code: '0x1::DaoVoteScripts::unstake_vote',
            type_args: ['0x1::STC::STC', '0x1::UpgradeModuleDaoProposal::UpgradeModuleV2'],
            args: [receiverAddressHex, '0']
          }
        }
        console.log('取回质押请求 -> ', JSON.stringify(txnRequest))
        const txnOutput = await provider.dryRun(txnRequest)
        console.log('取回质押执行模拟结果 -> ', txnOutput)

        // generate maxGasAmount from contract.dry_run -> gas_used
        const maxGasAmount = txnOutput.gas_used

        // because the time system in dev network is relatively static,
        // we should use nodeInfo.now_secondsinstead of using new Date().getTime()
        const nowSeconds = await provider.getNowSeconds()
        // expired after 12 hours since Unix Epoch
        const expirationTimestampSecs = nowSeconds + 43200

        // 生成原始提案交易数据
        const scriptFunction = this.buildUnstakeVoteScriptFunction(receiverAddressHex, 0)
        const rawVoteTransaction = this.generateRawUserTransaction(
          senderAddressHex,
          maxGasAmount,
          scriptFunction,
          senderSequenceNumber,
          expirationTimestampSecs,
          chainId
        )
        console.log('原始提案交易数据 -> ', rawVoteTransaction)
        // 签名和发送交易
        await this.signAndSendTransaction(senderPrivateKeyHex, rawVoteTransaction)

        this.$message({
          type: 'success',
          message: '取回质押成功!'
        })
        this.voteLoading = false
      } catch (error) {
        this.$message({
          type: 'error',
          message: '取回质押失败!' + error.message
        })
        this.voteLoading = false
      }
      await this.getAccountInfo()
    },
    async onSubmit(senderPrivateKeyHex, amount, agree) {
      this.voteLoading = true
      try {
        const provider = this.provider
        if (provider === null) {
          throw new Error('Starcoin 网络未准备好，请刷新页面重试')
        }
        const chainId = this.starcoinNetwork.chainId
        const senderAddressHex = this.accountInfo.address
        const senderPublicKeyHex = this.accountInfo.publicKey
        // senderAddressHex = '0x8b79fdf7bd004b72ea4bd83289429455'
        // 获取账号交易数
        const senderSequenceNumber = await provider.getSequenceNumber(senderAddressHex)

        // 提案发起者地址
        const receiver = '0xb2aa52f94db4516c5beecef363af850a'
        let receiverAddressHex = ''
        if (receiver.slice(0, 3) === 'stc') {
          const receiptIdentifier = starcoin.starcoin_types.ReceiptIdentifier.decode(receiver)
          receiverAddressHex = starcoin.encoding.addressFromSCS(receiptIdentifier.accountAddress)
        } else {
          receiverAddressHex = receiver
        }

        const sendAmountString = `${amount.toString()}u128`
        // 构建交易请求体
        const txnRequest = {
          chain_id: chainId,
          gas_unit_price: 1,
          sender: senderAddressHex,
          sender_public_key: senderPublicKeyHex,
          sequence_number: senderSequenceNumber,
          max_gas_amount: 10000000,
          script: {
            code: '0x1::DaoVoteScripts::cast_vote',
            type_args: ['0x1::STC::STC', '0x1::UpgradeModuleDaoProposal::UpgradeModuleV2'],
            args: [receiverAddressHex, '0', 'true', sendAmountString]
          }
        }
        console.log('提案请求 -> ', JSON.stringify(txnRequest))
        const txnOutput = await provider.dryRun(txnRequest)
        console.log('提案执行模拟结果 -> ', txnOutput)

        // generate maxGasAmount from contract.dry_run -> gas_used
        const maxGasAmount = txnOutput.gas_used

        // because the time system in dev network is relatively static,
        // we should use nodeInfo.now_secondsinstead of using new Date().getTime()
        const nowSeconds = await provider.getNowSeconds()
        // expired after 12 hours since Unix Epoch
        const expirationTimestampSecs = nowSeconds + 43200

        // 生成原始提案交易数据
        const voteScriptFunction = this.buildVoteScriptFunction(receiverAddressHex, 0, amount, agree)
        const rawVoteTransaction = this.generateRawUserTransaction(
          senderAddressHex,
          maxGasAmount,
          voteScriptFunction,
          senderSequenceNumber,
          expirationTimestampSecs,
          chainId
        )
        console.log('原始提案交易数据 -> ', rawVoteTransaction)
        // 签名和发送交易
        await this.signAndSendTransaction(senderPrivateKeyHex, rawVoteTransaction)

        this.$message({
          type: 'success',
          message: '投票成功!'
        })
        this.voteLoading = false
      } catch (error) {
        this.$message({
          type: 'error',
          message: '投票失败!' + error.message
        })
        this.voteLoading = false
      }
      await this.getAccountInfo()
    },
    /**
     * 构造 vote payload
     */
    buildVoteScriptFunction(receiver, proposalId, amount, agree) {
      let receiverAddress
      if (receiver.slice(0, 3) === 'stc') {
        const receiptIdentifier = starcoin.starcoin_types.ReceiptIdentifier.decode(receiver)
        receiverAddress = starcoin.encoding.addressFromSCS(receiptIdentifier.accountAddress)
      } else {
        receiverAddress = receiver
      }

      const functionId = '0x1::DaoVoteScripts::cast_vote'
      const tyArgs = [
        { Struct: { address: '0x1', module: 'STC', name: 'STC', type_params: [] }},
        { Struct: { address: '0x1', module: 'UpgradeModuleDaoProposal', name: 'UpgradeModuleV2', type_params: [] }}
      ]

      // Multiple BcsSerializers should be used in different closures, otherwise, the latter will be contaminated by the former.
      const amountSCSHex = (function() {
        const se = new starcoin.bcs.BcsSerializer()
        se.serializeU128(BigNumber(amount))
        return hexlify(se.getBytes())
      })()

      const proposalSCSHex = (function() {
        const se = new starcoin.bcs.BcsSerializer()
        se.serializeU64(proposalId)
        return hexlify(se.getBytes())
      })()
      const agreeSCSHex = (function() {
        const se = new starcoin.bcs.BcsSerializer()
        se.serializeBool(agree)
        return hexlify(se.getBytes())
      })()

      const args = [
        arrayify(receiverAddress),
        arrayify(proposalSCSHex),
        arrayify(agreeSCSHex),
        arrayify(amountSCSHex)
      ]

      const scriptFunction = starcoin.utils.tx.encodeScriptFunction(functionId, tyArgs, args)
      return scriptFunction
    },
    /**
     * 构造 unstake vote payload
     */
    buildUnstakeVoteScriptFunction(receiver, proposalId, amount, agree) {
      let receiverAddress
      if (receiver.slice(0, 3) === 'stc') {
        const receiptIdentifier = starcoin.starcoin_types.ReceiptIdentifier.decode(receiver)
        receiverAddress = starcoin.encoding.addressFromSCS(receiptIdentifier.accountAddress)
      } else {
        receiverAddress = receiver
      }

      const functionId = '0x1::DaoVoteScripts::unstake_vote'
      const tyArgs = [
        { Struct: { address: '0x1', module: 'STC', name: 'STC', type_params: [] }},
        { Struct: { address: '0x1', module: 'UpgradeModuleDaoProposal', name: 'UpgradeModuleV2', type_params: [] }}
      ]

      const proposalSCSHex = (function() {
        const se = new starcoin.bcs.BcsSerializer()
        se.serializeU64(proposalId)
        return hexlify(se.getBytes())
      })()

      const args = [
        arrayify(receiverAddress),
        arrayify(proposalSCSHex)
      ]

      const scriptFunction = starcoin.utils.tx.encodeScriptFunction(functionId, tyArgs, args)
      return scriptFunction
    },
    /**
     * 生成交易原始数据
     */
    generateRawUserTransaction(
      senderAddress,
      maxGasAmount,
      scriptFunction,
      senderSequenceNumber,
      expirationTimestampSecs,
      chainId) {
      // Step 1-2: generate RawUserTransaction
      const sender = starcoin.encoding.addressToSCS(senderAddress)
      const sequence_number = BigNumber(senderSequenceNumber)
      const payload = scriptFunction
      const max_gas_amount = BigNumber(maxGasAmount)
      const gas_unit_price = BigNumber(1)
      const gas_token_code = '0x1::STC::STC'
      const expiration_timestamp_secs = BigNumber(expirationTimestampSecs)
      const chain_id = new starcoin.starcoin_types.ChainId(chainId)

      const rawVoteTransaction = new starcoin.starcoin_types.RawUserTransaction(sender, sequence_number, payload, max_gas_amount, gas_unit_price, gas_token_code, expiration_timestamp_secs, chain_id)

      return rawVoteTransaction
    },
    /**
     * 签名并发送交易
     */
    async signAndSendTransaction(senderPrivateKeyHex, rawUserTransaction) {
      const signedUserTransactionHex = await starcoin.utils.tx.signRawUserTransaction(
        senderPrivateKeyHex,
        rawUserTransaction
      )

      console.log('提案交易数据签名 -> ', signedUserTransactionHex)

      // 发送交易
      const txn = await this.provider.sendTransaction(signedUserTransactionHex)
      console.log('提案交易执行结果 -> ', txn)

      const txnInfo = await txn.wait(1)
      console.log('提案交易确认结果 -> ', txnInfo)
    },
    onCancel() {
      this.$message({
        message: 'cancel!',
        type: 'warning'
      })
    }
  }
}
</script>

<style scoped>
.line{
  text-align: center;
}
.el-row {
  margin-bottom: 20px;
}
.el-col {
  border-radius: 4px;
}
</style>

