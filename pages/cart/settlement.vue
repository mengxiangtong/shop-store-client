<template>
  <div class="order">
    <van-nav-bar :z-index="10"
        @click-left="$router.go(-1)"
        fixed
        left-arrow
        left-text="返回"
        title="提交订单"></van-nav-bar>
    <nuxt-link class="address" tag="div" to="/address">
      <div class="address-detail">
        {{hasDefault ? addressInfo.name : "选择/添加收货地址"}}
      </div>
      <div class="address-info" v-if="hasDefault">{{addressInfo.addressDetail}}</div>
      <van-icon name="arrow"/>
      <van-icon name="location" v-if="hasDefault"/>
    </nuxt-link>
    <van-panel title="商品">
      <van-card
          :key="item.id"
          :num="item.count"
          :origin-price="formatPrice(item.price)"
          :price="formatPrice(item.salePrice)"
          :thumb="item.imageUrl"
          :title="item.name"
          v-for="item in settlementList">
        <div class="desc" slot="desc">
          {{item.description.length > 32 ? item.description.slice(0,32) + "..." : item.description}}
        </div>
      </van-card>
    </van-panel>
    <van-cell-group>
      <van-cell :label="labelString"
          :value="dealMoney ? '￥' + (dealMoney.toFixed(2)): '免运费'"
          title="运费"/>
      <van-field label="留言"
          placeholder="点击给买家留言"
          v-model="remark"/>
      <van-cell :value="'￥' + (totalPrice.toFixed(2))"
          style="color:#f44"
          title="合计"/>
    </van-cell-group>
    <div class="order-footer">购物愉快~</div>
    <van-submit-bar :price="totalPrice * 100"
        @submit="onSubmit"
        button-text="结算"/>
    <loading :loading="loading"></loading>
  </div>
</template>

<script>
  import {handleError} from "../../utils/utils";
  import Loading from "../../components/loading";

  export default {
    components: {Loading},
    data() {
      return {
        remark: "",
        labelString: "",
        settlementIds: [],
        settlementList: [],
        defaultAddress: {},
        loading: true,
        isBuyNow: this.$route.query.type == 1
      };
    },
    computed: {
      totalCartPrice() {
        return this.settlementList.reduce((t, c) => {
          return t + c.salePrice * c.count;
        }, 0);
      },
      totalPrice() {
        return this.totalCartPrice + this.dealMoney;
      },
      dealMoney() {
        return this.totalCartPrice >= 150 ? 0 : 6;
      },
      addressInfo() {
        return {
          name: this.defaultAddress.name,
          phone: this.defaultAddress.tel,
          addressDetail: this.defaultAddress.provinceName + " " + this.defaultAddress.cityName + " "
            + this.defaultAddress.countryName + " " + this.defaultAddress.detailAddress
        };
      },
      hasDefault() {
        return Object.keys(this.defaultAddress).length;
      }
    },
    mounted() {
      this.getSettlementList();
    },
    methods: {
      async getSettlementList() {
        let settlementList = localStorage.getItem("settlementList");
        let isSettlement = localStorage.getItem("isSettlement");
        if (settlementList && Object.prototype.toString.apply(JSON.parse(settlementList)) === "[object Array]" && JSON.parse(settlementList).length) {
          let arr = JSON.parse(settlementList);
          this.settlementIds = arr;
          if (isSettlement && JSON.parse(isSettlement)) {
            this.getDefaultAddress();
          } else {
            let selectAddress = localStorage.getItem("selectAddress");
            if (selectAddress) {
              this.defaultAddress = JSON.parse(selectAddress) || {};
            } else {
              this.getDefaultAddress();
            }
          }
          try {
            let url = "/cart/getCartById";
            let params = {ids: arr.join(",")};
            if (this.isBuyNow) {
              url = "book/getBookInfoById";
              params = {id: arr[0]};
            }
            let res = await this.$axios.$post(url, params);
            this.loading = false;
            if (res.errorCode === 200) {
              let data = res.data;
              if (this.isBuyNow) {
                data.count = parseInt(this.$route.query.count);
                this.settlementList = [data];
              } else {
                this.settlementList = res.data;
              }
            } else {
              this.$notify({
                message: res.errorMsg
              });
            }
          } catch (err) {
            handleError(err, this.$router);
            this.loading = false;
          }
        } else {
          this.$notify({
            message: "参数错误"
          });
          setTimeout(() => {
            this.$router.replace("/cartIndex");
          }, 1000);
        }
      },
      // 获取默认地址
      async getDefaultAddress() {
        try {
          let res = await this.$axios.$post("/address/getDefaultAddress");
          if (res.errorCode === 200) {
            this.defaultAddress = res.data || {};
          } else {
            this.$notify({
              message: res.errorMsg
            });
          }
        } catch (err) {
          handleError(err, this.$router);
        }
      },
      // 提交订单
      async onSubmit() {
        if (!this.defaultAddress || !this.defaultAddress.id) {
          this.$notify({
            message: "请选择收货地址"
          });
          return false;
        }
        try {
          let params = {
            deliveryAddressId: this.defaultAddress.id,
            ids: this.settlementIds.join(","),
            remark: this.remark,
            type: this.$route.query.type,
            count: this.$route.query.count
          };
          let res = await this.$axios.$post("/order/createdOrder", params);
          if (res.errorCode === 200) {
            this.$notify({
              message: "订单创建成功",
              duration: 500,
              background: "#1989fa"
            });
            setTimeout(() => {
              this.$router.push({
                path: "/safe/validPayPwd",
                query: {
                  orderId: res.data.orderId
                }
              });
            }, 500);
          } else {
            this.$notify({
              message: res.errorMsg
            });
          }
        } catch (err) {
          handleError(err, this.$router);
        }
      },
      formatPrice(price) {
        return price.toFixed(2);
      }
    },
    beforeRouteEnter(to, from, next) {
      if (from.path === "/safe/validPayPwd") {
        next({
          replace: true,
          path: "/cartIndex"
        });
      } else {
        next();
      }
    }
  };
</script>

<style lang="stylus" scoped>
  .order {
    height 100vh
    background-color #eee
  }

  .address {
    position relative
    padding-top 46px
    line-height 36px
    font-weight bold
    background-color #fff

    .address-detail {
      padding-left 34px
      font-size 16px
    }

    .address-info {
      font-size 13px
      line-height 1
      font-weight normal
      margin-bottom 10px
      padding-left 34px
      color #7F828B
    }

    &::after {
      content ''
      display block
      height 2px
      background -webkit-repeating-linear-gradient(135deg, #ff6c6c 0, #ff6c6c 20%, transparent 0, transparent 25%, #3283fa 0, #3283fa 45%, transparent 0, transparent 50%)
      background repeating-linear-gradient(-45deg, #ff6c6c 0, #ff6c6c 20%, transparent 0, transparent 25%, #3283fa 0, #3283fa 45%, transparent 0, transparent 50%)
      background-size 80px
    }

    .van-icon-arrow {
      position absolute
      right 10px
      top 57px
      font-weight bold
    }

    .van-icon-location {
      position absolute
      left 10px
      top 58px
      font-weight bold
    }
  }

  .van-panel {
    margin 12px 0
    padding-bottom 10px
  }

  .order-footer {
    margin-top 10px
    text-align center
    font-size 13px
    color #969799
  }
</style>
