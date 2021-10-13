# vue-virtual-scroller的使用，长列表优化，虚拟列表，纵想丝滑

对于长列表来说，我们大部分的操作都是：1.懒加载，分页，2.Object.freeze冻结数组取消响应式，因为大多时候都是展示   3.高清图替换成缩略图，因为很多时候长列表的图尺寸都比较小，所以可以用小图来代替

以上能解决大部分的长列表问题，在可以分页的情况下

但是，还有两个问题没有解决，那就是：1.不能分页的时候怎么办

2.当用户向下滑动加载了很多很多的内容时，可能是1000个10000个的时候。这个时候浏览器就会变得卡顿，特别是在手机上。原因就是因为浏览器渲染了太多的div，消耗了很多的资源（重绘和回流都是需要浏览器资源的）

所以，针对这两个问题，我们隆重的推出我们的虚拟列表，关于虚拟列表的原理，已经有很多很多的文章讲过了，本文就不再去重复了，大家可以直接去搜索一下就可以出来它的原理，大概就是只渲染缓冲区和可视区的盒子，不会去把所有列表中的盒子渲染出来，达到节省浏览器资源的目的。

另外的话，这个虚拟列表主要比较好用的组件库有 vue 中：vue-virtual-scroller 和vue-virtual-scroll-list

react的话 ，基本上就是 react-virtualized ，这个组件也非常的厉害，常用react的同学可以自己去研究下这个组件库



因为我自己是比较常用vue，下面的话，就是去教大家怎么去vue-virtual-scroller这个组件库来实现虚拟列表的

首先的话，应该去看一下官网：https://www.npmjs.com/package/vue-virtual-scroller#dynamicscroller

然后再main.js中导入，注册使用，千万记得要把样式导入进去，之前我忘记导入样式了，找了老半天才发现

```js
import 'vue-virtual-scroller/dist/vue-virtual-scroller.css'
import VueVirtualScroller from 'vue-virtual-scroller'
Vue.use(VueVirtualScroller)
```

然后，你就可以愉快的去使用里面的组件了



好的，让我们来实现第一个虚拟列表的demo，这个demo的话，是固定了每一个item的高度的，并且也是一次性就去加载数据的，不会有上拉加载数据的操作。也就是说，如果你要渲染的list就是这么多，并且高度一致的话，就可以用这个去改

代码：

```js
<template>
  <div class="home">
    <!-- <div class="father">
      <div class="right">
        <div class="every_type">
          <div class="item_box">
            <div
              class="item"
              v-for="(item, index) in items"
              :key="index"
            >
              {{ item.title }}
            </div>
          </div>
          
        </div>
      </div>
    </div> -->
    <RecycleScroller
      class="scroller"
      :items="items"
      :item-size="50"
      v-if="items.length"
    >
      <template v-slot="{ item }">
        <div class="user">
          {{ item.title }}
        </div>
      </template>
    </RecycleScroller>
  </div>
</template>

<script>
// @ is an alias to /src

export default {
  name: 'Home',
  components: {},
  data () {
    return {
      items: []
    }
  },
  created () {
    this.getData()
    // console.log(this.items)
  },
  methods: {
    getData () {
      for (let i = 0; i < 200000; i++) {
        this.items.push({ title: 'ssadqwesadwqe' ,id : i})
      }
    }
  }
}
</script>

<style lang="less" scoped>
.scroller {
  height: 300px;
  background-color: rgba(0, 0, 0, 0.1);
}

.user {
  height: 50px;
  padding: 0 12px;
  display: flex;
  align-items: center;
}
</style>

```

使用起来非常的简单，像这种只需要使用到RecycleScroller就行了，:items="items"设置的是对应的数组，:item-size="50"设置的是每个item的高度。另外需要注意的就是下面的那两个样式记得要复制到位，因为虚拟列表要求你的scroller和user是要有高度的

怎么判断我们的虚拟列表使用成功了呢？可以打开F12的element来看，当我们滚动滚动条，但是盒子不增加，只做位移操作的话，就可以判断我们的虚拟列表使用成功了。如果没有使用的虚拟列表的话，那里的item盒子应该是20w个

![虚拟列表](C:\Users\Administrator\Desktop\虚拟列表.jpg)

另外的话，我的代码里面还有一段注释，就是没有使用虚拟列表时去渲染20w个div时，大家可以打开注释来看一下，哪个时间花的比较多。其实我已经测过了，虚拟列表花的时间短多了！



已经知道了基本的使用，那么我们就可以加深一点，看一下，如果一次不加载这么多，只加载5个数据，然后上拉触底的时候再去加载，这种应该如何去实现。（这种还是高度一定，但是可以上拉加载了）

上代码

```js
<template>
  <div>
    <RecycleScroller
      class="scroller"
      :items="items"
      :item-size="300"
      :emitUpdate="true"
      @update="update"
      @resize="resize"
      @visible="visible"
      @hidden="hidden"
      @scroll="scroll"
      v-if="items.length"
    >
      <template slot-scope="props">
        <li :key="props.itemKey">
          <div>{{ props.item.title }}</div>
          <img :src="props.item.img" alt="" />
        </li>
      </template>
    </RecycleScroller>
  </div>
</template>
<script>
export default {
  name: 'test',
  data () {
    return {
      items: []
    }
  },
  created () {
    this.getData()
  },
  mounted () {},
  methods: {
    getData () {
      this.$axios('/home/swiper').then(res => {
        console.log(res)
        this.items = res.data.data.list
      })
    },
    scroll () {
      console.log(111)
    },
    update (start, end) {
      // console.log(start, end)
      if (end === this.items.length) {
        console.log(1111)
        let temp = []
        // temp.push({
        //   id: 101,
        //   img: '"http://dummyimage.com/200x100/FF6600"',
        //   time: '2003-02-02',
        //   title: 'hahahha'
        // })
        this.$axios('/home/add').then(res => {
          // console.log(res)
          // this.items = res.data.data.list
          temp = [...this.items, ...res.data.data.list]
          // console.log(temp)
          this.items = temp
        })
        // this.items = temp
      }
    },
    resize () {
      console.log('resize')
    },
    visible () {
      console.log('visible')
    },
    hidden () {
      console.log('hidden')
    }
  }
}
</script>
<style lang="css" scoped>
.scroller {
  height: 300px;
  background-color: #ccc;
}

.user {
  height: 32%;
  padding: 0 12px;
  display: flex;
  align-items: center;
}
</style>

```

这段代码里面稍微比刚才的复杂了一点点，其实就是加了:emitUpdate="true" 和 @update="update"。其实最重要的是这个update方法，那为什么需要:emitUpdate="true"呢，是因为人家官网说了：要想触发这个update事件，就必须去设置emitUpdate为true。

在这个update方法中，我们就只做了一件事，就是当我们的结束的index等于我们的数组长度的时候，我们就去请求接口来获取新的数据，然后加入到数组中就行了。



最后，肯定还有朋友遇到的是不定高度的盒子，并且还需要进行上拉加载。

果真产品思想不滑坡，困难总比方法多！

上代码：

```js
<template>
  <div>
    <DynamicScroller
      :items="items"
      :min-item-size="54"
      class="scroller"
      :emitUpdate="true"
      @update="update"
      @resize="resize"
      @visible="visible"
      @hidden="hidden"
      @scroll="scroll"
      v-if="items.length"
    >
      <template v-slot="{ item, index, active }">
        <DynamicScrollerItem
          :item="item"
          :active="active"
          :size-dependencies="[item.message]"
          :data-index="index"
        >
          <li class="single-item" :key="item.id">
            <div class="left-pic"><img :src="item.img" alt="" /></div>
            <div class="right-info">
              <span>标题：{{ item.title }}</span>
              <span>项目数量：{{ item.id }}</span>
              <span>项目时间:{{ item.time }}</span>
              <span>项目描述:{{ item.des }}</span>
            </div>
          </li>
        </DynamicScrollerItem>
      </template>
    </DynamicScroller>
  </div>
</template>
<script>
export default {
  name: 'test',
  data () {
    return {
      items: []
    }
  },
  created () {
    this.getData()
  },
  mounted () {},
  methods: {
    getData () {
      this.$axios('/home/swiper').then(res => {
        console.log(res)
        this.items = res.data.data.list
      })
    },
    scroll () {
      console.log(111)
    },
    update (start, end) {
      if (end === this.items.length) {
        console.log(1111)
        let temp = []
        this.$axios('/home/add').then(res => {
          temp = [...this.items, ...res.data.data.list]
          this.items = temp
        })
      }
    },
    resize () {
      console.log('resize')
    },
    visible () {
      console.log('visible')
    },
    hidden () {
      console.log('hidden')
    }
  }
}
</script>
<style lang="less" scoped>
.scroller {
  height: 300px;
  background-color: #ccc;
}

.user {
  height: 32%;
  padding: 0 12px;
  display: flex;
}
.single-item {
  display: flex;
  justify-content: flex-start;
  align-items: center;
  border-bottom: 1px solid rgb(187, 167, 167);
  .left-pic {
    width: 200px;
    img {
      width: 200px;
    }
  }
  .right-info {
    padding-left: 20px;
    text-align: left;
    span {
      display: block;
      &:last-child {
        word-break: break-all;
      }
    }
  }
}
</style>

```

这次使用的组件就是DynamicScroller和DynamicScrollerItem，因为第一个组件RecycleScroller官网也说了，只能用于固定高度的情况下。

这一块的话，其实最主要的就是替换了组件。删除了传入固定高度的代码。

以下是我的github地址，没看懂的可以运行项目来看看，应该很容易上手：https://github.com/rui-rui-an/virtual-scroller-demo
