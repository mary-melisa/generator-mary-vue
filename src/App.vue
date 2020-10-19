<template>
    <div id="app">
        <!-- <transition :name="animate"> -->
            <keep-alive>
                <router-view v-if="$route.meta.keepAlive" class="Router"></router-view>
            </keep-alive>
        <!-- </transition> -->
        <!-- <transition :name="animate"> -->
            <router-view v-if="!$route.meta.keepAlive" class="Router"></router-view>
        <!-- </transition> -->
    </div>
</template>

<script>
import utils from "@/utils";
import { Dialog, Notify } from "vant";
import { enums } from "@/constants/enum";
import bus from "@/business/globalApplication";
//import workerManager from "@/business/workerManager";
import * as userTypes from "@/constants/actions/user";
import * as accountSvc from "@/service/accountService";
import * as goodsTypes from "@/constants/actions/goods";
import * as orderTypes from "@/constants/actions/order";
import * as accoundTypes from "@/constants/actions/account";
import * as configTypes from "@/constants/actions/config";
import * as actionTypes from '@/constants/actionTypes';
import * as BulletinTypes from '@/constants/actions/bulletin';
import * as marketType from '@/constants/actions/market';
import { constants } from "@/business/constants/constant";
import { mapActions, mapMutations, mapGetters } from "vuex";
import SessionUtil from '@/utils/applicationStorage/sessionStorageUtil';

export default {
    data() {
        return {
            animate:''
        }
    },

    computed: {
        ...mapGetters({
            getLoadComplete: configTypes.loadComplete,
            getAccountInfo: accoundTypes.accountInfo, // 资金账号列表
        })
    },

    mounted() {
        // 加载项目依赖
        this.loadProjectDeps();
        // 接受通知
        this.initSystemNotice();
        //添加侦听事件
        window.addEventListener("load", this.refreshData, false);
        //网络由异常到正常时触发
        window.addEventListener('online',  function(){Notify({ type: 'success', message: '网络连接上了'})},false);
        //网络由正常常到异常时触发
        window.addEventListener("offline", () => {Notify({ type: 'danger', message: '您的网络不可用,请检查'})},false);       
    },
  
    methods: {
        ...mapMutations({
            setGoodsGroupList: goodsTypes.goodsGroupList,
            setUserInfo: accoundTypes.userInfo,
            setTaAccountInfoArr: accoundTypes.taAccountInfoArr,
            seTaAccountInfoAllArr: accoundTypes.taAccountInfoAllArr,
            setHolderSums: orderTypes.holderSums,
            setHolderDetails: orderTypes.holderDetails,
            setLoadComplete: configTypes.loadComplete,
            setMarketList: marketType.marketList,
            setPDInfo: marketType.PDInfo,                // 奖池信息
            setCurrencyGoodsInfo: marketType.currencyGoodsInfo,  // 货币商品信息（JD|PD）
            setGoodsConfig: "goodsConfig",
            setAuctionShop: marketType.auctionShop, // 店铺
            setTradeposition:"tradeposition",
        }),

        ...mapActions({
            Bulletin_GetBulletin: BulletinTypes.Bulletin_GetBulletin,        // 通知轮询
            loopPDInfo: marketType.PDInfo,
        }),

        // 加载项目依赖
        loadProjectDeps() {
            bus.initCoreManager();

            bus.commonManager.loadProto(
                "protoBuf",
                require("@/business/proto/WeMTP.proto")
            );

            bus.commonManager.loadProto(
                "extraBuf",
                require("@/business/proto/ServiceMemoryCache.proto")
            );

            //workerManager.initWorker();

            //this.initWorkerEvent();
        },

        // 刷新数据
        refreshData(reconnection = false) {
            if (this.$router.currentRoute.path !== "/login") {
                const loginRsp = bus.accountManager.loginRsp;
                if (loginRsp && SessionUtil.getItem('__loginToken__')) {
                    // 存在则表示已登录后刷新
                    this.setLoadComplete(false);
                    accountSvc
                        .systemRefresh()
                        .then(res => {
                            let {
                                markInfo,
                                authGoodsInfoList,
                                holdSumList,
                                PDInfo,
                                currencyGoodsInfo,
                                auctionShop,
                                goodsConfig,
                                tradeposition
                            } = res;
                            this.setMarketList(markInfo);
                            this.setCurrencyGoodsInfo(currencyGoodsInfo)
                            this.setAuctionShop(auctionShop);
                            // 缓存账户信息;
                            this.setUserInfo(bus.accountManager.moAccountInfo);
                            // 缓存过滤后的资金账户信息
                            this.setTaAccountInfoArr(bus.accountManager.moTaAccountInfo);
                            // 缓存所有资金账户信息
                            this.seTaAccountInfoAllArr(
                                bus.accountManager.taAccountList
                            );
                            this.setPDInfo(PDInfo);       // 奖池信息
                            this.loopPDInfo();            // 定期刷新奖池信息
                            
                            bus.stateManager.isLogin = true;
                            // 缓存持仓汇总
                            this.setHolderSums(bus.orderManager.holderSums);
                            // 缓存持仓明细
                            this.setHolderDetails(bus.orderManager.holderDetails);

                            // 缓存商品信息
                            this.setGoodsGroupList(authGoodsInfoList);                        
                            this.Bulletin_GetBulletin();    // 通知轮询

                            this.setGoodsConfig(goodsConfig);
                            this.setTradeposition(tradeposition);

                            // 计算盈亏
                            bus.orderManager.calcHolderSumAll();
                            bus.accountManager.calcAmount();

                            this.setLoadComplete(true);
                        })
                        .catch(err => {
                            console.error(err);
                            this.$loading.hide();
                            // 停止交易链路的心跳，断开网络
                            bus.stateManager.deallocObject();   
                            // 停止行情链路的心跳
                            //workerManager.stopHeartbeat();
                            //跳转至登录页
                            this.$router.replace("/login");
                            SessionUtil.setItem('__loginToken__', false);
                            this.loginErr(reconnection);
                        });
                } else {
                    // Notify({ type: 'danger', message: '您目前不是登录状态,请登录'})
                    this.$route.meta.isLogin = false;
                    SessionUtil.setItem('__loginToken__', false);
                    this.$router.replace("/login");
                }
            }
        },

        // 系统级通知
        initSystemNotice() {
            // 离线通知
            bus.broadcastManager.subscribe(constants.Action.Action_LogoutRsp,response => {
                    // 停止交易链路的心跳，断开网络
                    bus.stateManager.deallocObject();   
                    // 停止行情链路的心跳(行情走的是worker 而交易没有)
                    //workerManager.stopHeartbeat();                
                    Dialog.alert({
                        className: 'logoutDialog',
                        title: "温馨提示",
                        message: `您的账号已经登出${response.msg}`
                    }).then(() => {
                        this.backToLogin('离线通知');
                    });
            });

            // 断网重连状态
            bus.broadcastManager.subscribe(constants.Action.Action_Network_OnReconnectChangeState, (sender) => {
                // BeginReconnect: 0,        // 开始重连
                // FailAndWaitPeriod: 1,     // 尝试重连失败，将在下一个周期后再次尝试重连
                // ReconnectSuccessed: 2,    // 重连成功，将进行业务操作
                // LoginFail: 3,             // 重连成功后业务操作失败并将再次重试，由业务模块发起
                // Logined: 4                // 重连成功后业务操作成功（包括交易服务的账户登录状态更新或行情服务的行情订阅等
                if (sender.msg && sender.msg.type ===1 && sender.msg.state === enums.ReconnectChangeState.ReconnectSuccessed) {
                    bus.accountManager.tokenCheck().then(()=>{
                        // Notify({ type: 'success', message: '服务器连接成功'});
                    },(err)=>{
                        this.refreshData(true);
                    })
                }
            });

            //系统结算
            bus.broadcastManager.subscribe(constants.Action.Action_MarketSettleChangeNotify, () => {
                Dialog.alert({
                    className: "logoutDialog",
                    title: '系统通知',
                    message: '系统结算，请重新登录'
                }).then(() => {
                    this.backToHome();
                });
            });

            // 订阅资金变化通知
            bus.broadcastManager.subscribe(constants.Action.Action_MoneyChangedNtf, () => {
                this.setTaAccountInfoArr(bus.accountManager.moTaAccountInfo);
            }, this);

            // 接收头寸变化通知
            bus.broadcastManager.subscribe(constants.Action.Action_PosChangedNtf, async (arg1) => {
                //查询持仓头寸表
                bus.auctionManager.querySZDZ2_Tradeposition({accountId : String(this.getAccountInfo.AccountId)}).then(res => {
                    if(res && res.length) this.setTradeposition(res);
                }).catch(err=>{
                    console.error('接收头寸变化通知时,查询头寸数据(拍点,金点)报错');
                });
            });
        },

        //登录异常,重刷
        loginErr(reconnection = false){
            Dialog.alert({
                className: 'logoutDialog',
                title: "提示",
                message: reconnection ? '您离开太久,请重新登录' : '加载超时,请重新登录'
            }).then(() => {
                window.location.reload();
            });
        },

        // 返回登录页面
        backToLogin (err) {
            //若果err是一个数字 则说明是一个步骤出错 需要作出提示
            if(!isNaN(err)) Notify({ type: 'danger', message: `加载失败,请稍后重试(${err})`});
            //控制台打印数据
            console.error(err);
            this.$loading.hide();
            //加载失败 相关操作置为false
            this.setLoadComplete(false);
            SessionUtil.setItem('__loginToken__', false);
            this.$route.meta.isLogin = false;
            // 停止交易链路的心跳，断开网络
            bus.stateManager.deallocObject();   
            // 停止行情链路的心跳
            //workerManager.stopHeartbeat();
            //跳转至登录页
            this.$router.replace("/login");
            window.location.reload();
        },

        // 返回首页
        backToHome () {
            bus.stateManager.isLogin = false;
            SessionUtil.setItem('__loginToken__', false);
            this.$router.replace('/login');
            window.location.reload();
        },

        initWorkerEvent() {
            //金点拍专用行情推送
            //bus.broadcastManager.publish(constants.Action.Action_SZJDPQuote, list);
            // workerManager.recvTikFun = function(list) {
            //     //金点拍专用行情推送
            //     bus.broadcastManager.publish(constants.Action.Action_SZJDPQuote, list);
            // };
        },

        //轮询通知公告
        getBulletin(){
            bus.bulletinManager.newNoticeID = bus.accountManager.serverLastNoticeId;
            this.Bulletin_GetBulletin();
        },
    },

    watch: {
        //监听是否初始化完成了
        getLoadComplete(val){
            if(val) this.getBulletin();
        },
        // $route(to, from) {
        //     try{
        //         //设置动画名称
        //         if(to.meta.index === from.meta.index){
        //             this.animate= '';
        //         }else if(to.meta.index < from.meta.index){
        //             this.animate= 'van-slide-left';
        //         }else if(to.meta.index > from.meta.index){
        //             this.animate= 'van-slide-right';
        //         }else{
        //             this.animate= '';
        //         }
        //     }catch(err){
        //         this.animate= '';
        //     }
        // }
    }
};
</script>

<style lang="scss">
#app {
    width: 100%;
    height: 100%;
}
.tabbar,
.main {
    width: 100%;
    height: 100%;
    min-height: 100%;
    overflow: hidden;
}
.Router {
    position: absolute !important;
    min-height: 100vh !important;
    background-color: #fff;
    width: 100% !important;
}
</style>