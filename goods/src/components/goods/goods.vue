<template>
  <div class="goods">
    <v-brumbs>商品</v-brumbs>
    <div class="w1260">
      <div class="sort-wrapper">
        <span class="type">排序方式：</span>
        <a href="#" @click.prevent="sortCh(0)" :class="{'active': sortType === 0}">默认排序</a>
        <a href="#" @click.prevent="sortCh(sortType === 0 ? 1 : sortType === 1 ? -1 : 1)"
           :class="{'active': sortType !== 0, 'desc-bottom': sortType === -1}">
          价格<i class="icon-price"></i></a>
        <span class="order-price" @click="filterShow=true">价格筛选</span>
      </div>
      <div class="goods-wrapper">
        <transition name="bg">
          <div class="bg" @click="filterShow=false" v-show="filterShow"></div>
        </transition>
        <transition name="filter">
          <dl class="price-list" v-show="!isMobile || (isMobile && filterShow)">
            <dt class="title">价格区间：</dt>
            <dd class="item" :class="{'active': priceSelect === -1}" @click="priceRing(-1)">ALL</dd>
            <dd class="item" v-for="(item, index) in priceFilter" @click="priceRing(index)"
                :class="{'active': index === priceSelect}">
              {{ item.start.toFixed(2) }} - {{ item.end.toFixed(2) }}
            </dd>
          </dl>
        </transition>
        <ul class="goods-list">
          <li class="item" v-for="(item, index) in goods" :key="index">
            <a href="#" @click.prevent="" class="image">
              <img border="0" v-lazy="'/static/images/'+item.productImage">
            </a>
            <b class="name">{{ item.productName }}</b>
            <span class="price">￥ {{ formatM(item.salePrice * 1) }}</span>
            <a class="add-car" href="#" @click.prevent="shopCar(item.productId)">加入购物车</a>
          </li>
        </ul>
        <transition name="load">
          <div v-show="loading" class="loading-more" v-infinite-scroll="loadMore" infinite-scroll-disabled="busy"
               infinite-scroll-distance="10">
          </div>
        </transition>
      </div>
    </div>
    <v-confirm
      ref="confirm"
      msg="加入购物车成功!"
      :btns="confirmBtns"
      @leftClick="confirmHide"
      @rightClick="goShopcar"></v-confirm>
  </div>
</template>

<script type="text/ecmascript-6">
  import axios from 'axios';
  import { mapGetters, mapMutations } from 'vuex';
  import Brumbs from '@/base/crumbs/crumbs';
  import Confirm from '@/base/confirm/confirm';
  import { formatMoney } from '@/common/js/format';

  const SORT_DEFAULT = 0;
  const SELECT_ALL = -1;

  export default {
    data () {
      return {
        goods: [],
        priceFilter: [
          {
            start: 0,
            end: 1000
          },
          {
            start: 1000,
            end: 2000
          },
          {
            start: 2000,
            end: 3000
          }
        ],
        priceSelect: SELECT_ALL,
        sortType: SORT_DEFAULT,
        page: 1,
        busy: false,
        loading: true,
        confirmBtns: {
          leftText: '继续购物',
          rightText: '查看购物车'
        },
        filterShow: false,
        widWidth: 0
      };
    },
    components: {
      'v-brumbs': Brumbs,
      'v-confirm': Confirm
    },
    created () {
      this._getGoodsList();
      window.onresize = () => {
        this.widWidth = window.innerWidth;
      };
    },
    computed: {
      isMobile () {
        if (this.widWidth === 0) {
          this.widWidth = window.innerWidth;
        }

        return this.widWidth <= 767;
      },
      ...mapGetters([
        'user'
      ])
    },
    methods: {
      sortCh (sort) {
        this.sortType = sort;
        this._getGoodsList(false, 1);
      },
      priceRing (index) {
        let flag = this.priceSelect === index;
        this.priceSelect = index;
        this._getGoodsList(flag, 1);
        this.filterShow = false;
      },
      shopCar (productId) {
        axios.post('/apis/goods/addCar', {
          userId: this.user.userId,
          productId
        }).then((res) => {
          res = res.data;
          if (res.status === 0) {
            this.setCartList(res.result.list);
            this.$refs.confirm.show();
          } else {
            this.setAlert({
              show: true,
              msg: '你当前未登录!'
            });
          }
        });
      },
      confirmHide () {
        this.$refs.confirm.hide();
      },
      goShopcar () {
        this.$router.push('/cart');
      },
      loadMore () {
        this.busy = true;
        setTimeout(() => {
          this.page++;
          this._getGoodsList(true);
        }, 500);
      },
      formatM (num) {
        return formatMoney(num);
      },
      _getGoodsList (isPush, pageSet) {
        let params = {
          size: 12
        };

        if (this.sortType !== SORT_DEFAULT) {
          params.sort = this.sortType;
        }

        if (pageSet) {
          this.page = pageSet;
        }

        if (this.priceSelect !== SELECT_ALL) {
          let priceCurrent = this.priceFilter[this.priceSelect];
          params.ring = {
            start: priceCurrent.start,
            end: priceCurrent.end
          };
        }

        params.page = this.page;
        axios.post('/apis/goods', {
          params
        }).then((res) => {
          if (res.data.status * 1 === 0) {
            if (isPush) {
              this.goods = this.goods.concat(Array.from(res.data.result.list));
              if (res.data.result.list.length === 0) {
                this.busy = true;
                this.loading = false;
              } else {
                this.busy = false;
                this.loading = true;
              }
            } else {
              this.goods = res.data.result.list;
              this.busy = false;
              this.loading = true;
            }
          }
        });
      },
      ...mapMutations({
        setCartList: 'SET_CART_LIST',
        setAlert: 'SET_ALERT'
      })
    }
  };
</script>

<style lang="less" scoped>
  @import url('../../common/style/basic.less');

  .goods {
    .w1260 {
      padding-top: 60px;

      .sort-wrapper {
        padding-right: 50px;
        height: 55px;
        font-size: 0;
        text-align: right;
        line-height: 55px;
        background-color: #fff;

        & > span {
          font-size: 14px;
          color: #605f5f;
        }

        & > a {
          margin-right: 15px;
          font-size: 14px;
          color: #605f5f;

          &.active,
          &:hover {
            color: #ee7a23;

            .icon-price {
              background-image: url(~"./icon-arrow-top-active.png");
            }
          }

          &.desc-bottom {
            .icon-price {
              transform: rotate(180deg);
            }
          }

          &:last-child {
            margin-right: 0;
          }

          .icon-price {
            display: inline-block;
            margin-left: 3px;
            width: 11px;
            height: 11px;
            background-image: url(~"./icon-arrow-top.png");
            background-size: cover;
          }
        }

        .order-price {
          display: none;
        }
      }

      .goods-wrapper {
        display: flex;
        flex-wrap: wrap;
        padding-top: 30px;

        .bg {
          display: none;
        }

        .price-list {
          flex: 0 0 230 / 1260 * 100%;
          padding: 0 20px;
          width: 230 / 1260 * 100%;
          font-size: 14px;
          color: #605f5f;

          .title {
            margin-bottom: 30px;
            font-weight: 700;
          }

          .item {
            margin: 20px 0;
            padding: 5px 0;
            cursor: pointer;
            transition: padding .5s ease-out,
            color .5s ease-out;

            &.active,
            &:hover {
              border-left: 2px solid rgb(238, 122, 35);
              padding-left: 15px;
              color: rgb(238, 122, 35);
            }
          }
        }

        .goods-list {
          padding: 0;
          flex: 1;
          font-size: 0;

          .item {
            display: inline-block;
            margin-bottom: 1.5%;
            margin-left: 1.5%;
            border: 2px solid #e9e9e9;
            width: 23.875%;
            font-size: 16px;
            background-color: #fff;
            box-sizing: border-box;
            transition: all .5s ease-out;

            .image {
              display: block;
              position: relative;
              width: 100%;
              padding-top: 100%;

              img {
                position: absolute;
                top: 0;
                left: 0;
                width: 100%;
                height: 100%;
              }
            }

            &:hover {
              border-color: #ee7a23;
              transform: translateY(-5px);
              box-shadow: 0 0 10px #999;
            }

            .name {
              display: block;
              overflow: hidden;
              margin: 20 / 236 * 100% 10 / 236 * 100% 0;
              height: 4em;
              line-height: 4/3em;
              font-size: 14px;
              color: #605f5f;
            }

            .price {
              display: block;
              margin: 0 10 / 236 * 100%;
              font-size: 17px;
              color: #d1434a;
            }

            .add-car {
              display: block;
              border: 1px solid #d1434a;
              margin: 15 / 236 * 100% 10 / 236 * 100%;
              height: 40px;
              font-weight: 700;
              text-align: center;
              line-height: 40px;
              color: #d1434a;
              transition: all .5s ease-out;

              &:hover {
                background-color: #ffe5e6;
              }
            }
          }
        }

        .loading-more {
          overflow: hidden;
          padding-left: 270px;
          width: 100%;
          height: 80px;
          box-sizing: border-box;
          transition: height .4s;

          &.load-leave-to,
          &.load-enter {
            height: 0;
          }

          &:after {
            display: block;
            margin: 30px auto 10px;
            width: 40px;
            height: 40px;
            background-image: url('/static/images/icon-loading.png');
            animation: roll 3s infinite;
            content: '';
          }

          @keyframes roll {
            0% {
              transform: rotate(0);
            }
            0% {
              transform: rotate(-360deg);
            }
          }
        }
      }

      @media screen and (max-width: 990px) {
        .goods-wrapper {
          .goods-list {
            .item {
              width: 95.5 / 300 * 100%;

              &:nth-of-type(3n + 1) {
                margin-left: 0;
              }
            }
          }
        }
      }

      @media screen and (min-width: 991px) {
        .goods-wrapper {
          .item:nth-of-type(4n + 1) {
            margin-left: 0;
          }
        }
      }

      @media screen and (max-width: 767px) {
        padding-top: 20px;

        .sort-wrapper {
          padding-left: 15px;
          padding-right: 15px;
          text-align: left;

          .type {
            display: none;
          }

          .order-price {
            display: block;
            float: right;
          }
        }

        .goods-wrapper {
          padding-top: 15px;

          .bg {
            display: block;
            position: fixed;
            top: 0;
            right: 0;
            bottom: 0;
            left: 0;
            z-index: 9998;
            background-color: rgba(0, 0, 0, 0.5);
            transition: all .5s;

            &.bg-enter,
            &.bg-leave-to {
              opacity: 0;
            }
          }

          .price-list {
            position: fixed;
            top: 0;
            right: 0;
            z-index: 9999;
            margin: 0;
            padding: 0;
            width: 60%;
            height: 100%;
            background-color: #fff;
            transition: all .5s;

            &.filter-enter,
            &.filter-leave-to {
              transform: translateX(100%);
            }

            .title {
              margin: 0;
              padding-left: 15px;
              width: 100%;
              height: 55px;
              line-height: 55px;
              background-color: #f0f0f0;
              box-sizing: border-box;
            }

            .item {
              margin: 0;
              border-bottom: 1px solid #e9e9e9;
              padding: 12px 10px 12px 15px;
              width: 100%;
              background-color: #fff;
              box-sizing: border-box;
            }
          }

          .goods-list {
            .item {
              position: relative;
              margin-right: 0;
              margin-left: 0;
              border-width: 1px 0;
              padding: 10px;
              width: 100%;

              &:hover {
                border-color: #e9e9e9;
                transform: none;
                box-shadow: none;
              }

              .image {
                float: left;
                border: 1px solid #e9e9e9;
                padding: 0;
                width: 108px;
                height: 108px;
              }

              .name {
                margin: 0 0 0 115px;
                font-size: 13px;
              }

              .price {
                margin: 0 0 0 115px;
                font-size: 13px;
              }

              .add-car {
                position: absolute;
                right: 10px;
                bottom: 10px;
                margin: 0;
                width: 87px;
                height: 28px;
                line-height: 28px;
                font-size: 12px;
              }
            }
          }

          .loading-more {
            padding: 0;
          }
        }
      }
    }
  }
</style>
