<template>
  <div>
    <button @click="autoMove">{{autoMoveTimer ? '停止自动移动' : '自动移动'}}</button>

    <button @click="move">手动移动</button>

    <ul style="margin-top: 50px;" ref="ul">
      <li ref="li"
          :data-index="index"
          :class="{active: moveIndex === index || toIndex === index, close: closeIndex === index, init: initIndex === index}"
          v-for="(item, index) in listData"
          :key="closeIndex === index ? Date.now() : item">
        <img :src="item">
      </li>
    </ul>
  </div>
</template>

<script>
  export default {
    data() {
      // MOCK列表
      const listData = ['https://ss0.bdstatic.com/70cFuHSh_Q1YnxGkpoWK1HF6hhy/it/u=1705832215,2808029359&fm=27&gp=0.jpg', 'https://ss2.bdstatic.com/70cFvnSh_Q1YnxGkpoWK1HF6hhy/it/u=1810152264,2923293270&fm=27&gp=0.jpg', 'https://ss2.bdstatic.com/70cFvnSh_Q1YnxGkpoWK1HF6hhy/it/u=3146830679,2953976497&fm=27&gp=0.jpg', 'https://ss2.bdstatic.com/70cFvnSh_Q1YnxGkpoWK1HF6hhy/it/u=1486060436,2399611194&fm=27&gp=0.jpg', 'https://ss1.bdstatic.com/70cFvXSh_Q1YnxGkpoWK1HF6hhy/it/u=1690728803,2100828734&fm=27&gp=0.jpg', 'https://ss0.bdstatic.com/70cFvHSh_Q1YnxGkpoWK1HF6hhy/it/u=4219344775,3123310119&fm=27&gp=0.jpg', 'https://ss1.bdstatic.com/70cFvXSh_Q1YnxGkpoWK1HF6hhy/it/u=2094859318,3382038355&fm=27&gp=0.jpg', 'https://ss2.bdstatic.com/70cFvnSh_Q1YnxGkpoWK1HF6hhy/it/u=2107823665,616413361&fm=27&gp=0.jpg', 'https://ss0.bdstatic.com/70cFvHSh_Q1YnxGkpoWK1HF6hhy/it/u=2136162895,4003705239&fm=27&gp=0.jpg', 'https://ss0.bdstatic.com/70cFvHSh_Q1YnxGkpoWK1HF6hhy/it/u=1451718319,9889403&fm=27&gp=0.jpg'].slice(0, 20);

      return {
        listData,          // 列表的数据
        moveIndex: -1,     // 交换前排名较高的索引
        toIndex: -1,       // 交换前排名较低的索引
        initIndex: -1,     // 即将添加展开通话的DOM索引
        closeIndex: -1,    // 即将添加关闭动画的DOM索引
        autoMoveTimer: null
      }
    },

    methods: {
      /*
      * 随机生成两个DOM索引
      * */
      selectDom() {
        const aIndex = Math.floor(Math.random() * this.listData.length);
        let bIndex;
        do {
          bIndex = Math.floor(Math.random() * this.listData.length);
        } while(aIndex === bIndex);

        if (aIndex > bIndex) {
          this.moveIndex = aIndex;
          this.toIndex = bIndex;
        } else {
          this.moveIndex = bIndex;
          this.toIndex = aIndex;
        }
      },

      /*
      * 开始进行过渡演示
      * */
      move() {
        if (this.moveTimer) {
          // 清除定时器
          clearTimeout(this.moveTimer);
          clearTimeout(this.clearTimer);
          this.moveTimer = null;
          this.clearTimer = null;

          // 如果还没开始移动就开始下次移动则直接展示上次最终状态
          this.remove();
        }

        // 使用nextTick是为了避免删除元素后计算位置错误
        this.$nextTick(() => {
          // 随机选中两个DOM
          this.selectDom();
          const { moveIndex } = this;

          // 克隆要移动的DOM
          const cloneDom = this.cloneDom();
          this.lastAppendChild = cloneDom;
          cloneDom.moveIndex = moveIndex;

          this.moveTimer = setTimeout(() => {
            // 开始进行移动
            this.moveDom(cloneDom);

            this.clearTimer = setTimeout(() => {
              this.clear(cloneDom);
              this.clearTimer = null;
              this.moveTimer = null;
            }, 300)
          }, 500)
        })
      },

      /*
      * 自动移动
      * */
      autoMove() {
        if (this.autoMoveTimer) {
          clearInterval(this.autoMoveTimer);
          this.autoMoveTimer = null;
          return;
        }

        this.autoMoveTimer = setInterval(() => {
          this.move();
        }, 1000)
      },

      /*
      * 克隆要移动的DOM
      * @returns {HTMLElement} 克隆的DOM
      * */
      cloneDom() {
        // 排名低的需要向上移动，克隆一个DOM
        const moveEl = document.querySelector(`[data-index='${this.moveIndex}']`);  // 需要移动的DOM
        const cloneDom = moveEl.cloneNode(true);

        cloneDom.style.background = 'yellow';
        cloneDom.style.position = 'fixed';
        cloneDom.style.left = 0;
        cloneDom.removeAttribute('data-index');

        // 计算top
        cloneDom.style.top = this.calcTop(moveEl) + 'px';
        this.$refs.ul.appendChild(cloneDom);
        // 缓存
        this.lastAppendChild = cloneDom;

        // 排名较高的DOM
        const toDom = document.querySelector(`[data-index='${this.toIndex}']`);
        // 计算目标的top
        cloneDom.toTop = toDom.offsetTop;
        cloneDom.toTop = this.calcTop(toDom);

        return cloneDom;
      },

      /*
      * 移动克隆的Dom
      * @param {HTMLElement} cloneDom 要移动的Dom
      * */
      moveDom(cloneDom) {
        // 开始进行移动
        cloneDom.style.top = cloneDom.toTop + 'px';

        // 在目标位置创建要移动的盒子
        this.addDom();
        // 做一个标识, 避免重复创建
        cloneDom.created = true;

        // 给这个盒子一个动画状态
        this.initIndex = this.toIndex;

        // 被移动的位置原位置收缩
        this.closeIndex = this.moveIndex + 1;    // 上方已经创建了一个盒子所以索引应该+1
        cloneDom.moveIndex++;

        // 清除移动盒子的选中状态
        this.moveIndex = -1;
      },

      /*
      * 将数据放到目标位置上
      * */
      addDom() {
        const moveContent = this.listData[this.moveIndex];
        this.listData.splice(this.toIndex, 0, moveContent);
      },

      /*
      * 移除多余元素, 展示最终列表
      * @param {HTMLElement|undefined} cloneDom 要移动的Dom
      * */
      clear(cloneDom) {
        // 移除克隆元素
        if (!cloneDom) {
          cloneDom = this.lastAppendChild;
          this.lastAppendChild = null;
        }

        if (!cloneDom.removed) {
          this.$refs.ul.removeChild(cloneDom);
          cloneDom.removed = true;
        } else {
          console.log('元素不存在')
        }

        // 移除移动盒子原位置的Dom
        this.listData.splice(cloneDom.moveIndex, 1);

        this.toIndex = -1;
        this.closeIndex = -1;
        this.initIndex = -1;
        this.moveIndex = -1;  // 正常情况下这个变量已经被重置, 但如果在连续点击的情况下将不会执行moveDom函数所以这里还是需要再次重置
      },

      /*
      * 移除上次未完成的动画元素
      * remove与clear的区别：remove用于动画非自然完成情况下, clear用于动画自然完成下
      * */
      remove() {
        // 创建新元素
        let moveIndex = this.lastAppendChild.moveIndex;
        if (!this.lastAppendChild.created) {
          this.addDom();
          moveIndex++;
        }
        // 移除克隆DOM
        this.$refs.ul.removeChild(this.lastAppendChild);
        // 移除原位置DOM
        this.listData.splice(moveIndex, 1);

        // 重置相关变量
        this.toIndex = -1;
        this.closeIndex = -1;
        this.initIndex = -1;
        this.moveIndex = -1;  // 正常情况下这个变量已经被重置, 但如果在连续点击的情况下将不会执行moveDom函数所以这里还是需要再次重置
        this.lastAppendChild = null;
      },

      /*
      * 计算盒子当前距离body的距离
      * @param {HTMLElement} ele 要计算位置的盒子
      * @returns {number} 当前位置距离body的距离
      * */
      calcTop(ele) {
        let offsetTop = ele.offsetTop;
        let offsetParent = ele.offsetParent;

        while (offsetParent) {
          offsetTop += offsetParent.offsetTop;
          offsetParent = offsetParent.offsetParent;
        }

        return offsetTop - (document.body.scrollTop || document.documentElement.scrollTop)
      }
    }
  }
</script>

<style lang="less">
  * {
    padding: 0;
    margin: 0;
  }
  ul, li {
    list-style: none;
  }

  @animationTime: 0.3s;
  @liHeight: 60px;

  ul {
    font-size: 13px;
    text-align: center;

    li {
      width: 100%;
      height: @liHeight;
      line-height: @liHeight;
      transition: background-color 0.3s linear, top @animationTime linear;
      position: relative;
      background: #FFF;
      overflow: hidden;

      &.active {
        background: yellow;
      }

      &.init {
        visibility: hidden;
        animation: openByHeight @animationTime linear;
      }

      &.close {
        visibility: hidden;
        animation: closeByHeight @animationTime linear forwards;
      }

      .clone {
        position: fixed;
        top: 0;
        left: 0;
      }

      img {
        width: 50px;
        height: 50px;
        border-radius: 50%;
      }

      span {
        margin-left: 10px;
        font-size: 16px;
        color: #1c23a5;
      }
    }
  }

  button {
    display: block;
    width: 200px;
    height: 50px;
    background: rgba(76, 141, 188, 0.2);
    margin: 10px auto;
    border: none;
  }

  @keyframes openByHeight {
    0% {
      height: 0;
    }

    to {
      height: @liHeight;
    }
  }

  @keyframes closeByHeight {
    0% {
      height: @liHeight;
    }

    to {
      height: 0;
    }
  }
</style>
