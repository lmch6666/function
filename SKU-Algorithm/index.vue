<script setup>
import { ref, reactive, computed } from "vue";
let goodsname = '爱疯4s'
let goodsid = "3893e02183901231123"
let color = ["黄色", "蓝色", "红色"];
let size = ["XL", "L"];
let type = ["儿童装", "女裤", "男裤"];
let typename = {
  "0": '颜色',
  '1': '款式',
  '2': '规格'
}
// 计算商品类型组合
function computeGoodsCount(list) {
  let res = [];
  let len = list.length;

  function helper(idx, prearr) {
    let ele = list[idx];
    let islast = idx >= list.length - 1;

    for (const item of ele) {
      let i = prearr.concat(item);
      if (islast) {
        res.push(i);
      } else {
        helper(idx + 1, i);
      }
    }
  }
  helper(0, []);
  return res;
}

let skulist = computeGoodsCount([color, type, size]);

// 计算属性
function computespecsLen(list) {
  return list.reduce((pre, cur) => {
    return [...pre, ...cur];
  }, []);
}
// 属性规格的集合
let specs = computespecsLen([color, type, size]);
// 邻接矩阵(这个其实是一个一维数组, 内部用了矩阵的点位置映射成数组的索引)
let Matrix = Array(specs.length * specs.length).fill(0);

// 选择的属性规格
let selectSpecs = ref([]);

function initMatrix(skulist) {
  skulist.forEach((sku, index) => {
    // 单个sku数据
    fillinMatrix(sku, index + 2);
  });
}

// 填充矩阵
function fillinMatrix(list, weight) {
  list.forEach((item) => {
    setadjoinMatrix(item, list, weight);
  });
}
 /** 
  * @item 列名
  * @skulist 行属性
  * @weight 权重
 */
function setadjoinMatrix(item, skulist, weight) {
  let idx1 = "";
  // 找当前属性在数组中得位置 列坐标
  for (let i = 0; i < specs.length; i++) {
    if (item === specs[i]) {
      idx1 = i;
      break;
    }
  }
  // 遍历sku每一个属性在数组中的位置 行
  skulist.forEach((item) => {
    let idx2 = "";
    for (let i = 0; i < specs.length; i++) {
      if (specs[i] == item) {
        idx2 = i;
        break;
      }
    }
    // 矩阵的点位置映射成数组的索引 
    // 当前属性在属性数组中的位置 * 属性数量 + 当前属性在sku组合中的位置
    // Matrix[idx1 * specs.length + idx2] = 1;
    let cur = Matrix[idx1 * specs.length + idx2]
    // 加上权重值 作用: 同个 sku 下连通 填充 同个数字。避免实际上不存在的sku的组合
    if(typeof cur !== 'number'){
       Matrix[idx1 * specs.length + idx2].push(weight)
    }else if(cur > 1){
       Matrix[idx1 * specs.length + idx2] = [cur, weight]
    }else {
       Matrix[idx1 * specs.length + idx2] = weight
    }
  });
}
 // 得到每一行格子的和 ,然后放进一个数组
function getColSum(params) {
  // 最终转化成矩阵式数据
  let col = params.map((item) => getVertexCol(item));
  // console.log(col);
  let sum = [];
  // 拿出每一列数据进行计算
  specs.forEach(( _, index ) => {
    let rowSum = col
      .map((item) => item[index])
      .reduce((pre, cur) => {
        let num = pre + (cur || 0);
        return num;
      }, 0);
    sum.push(rowSum);
  });
  return sum;
}
// 获取某一规格所在的那一列
function getVertexCol(specguige) {
  let idx = "";
  for (let i = 0; i < specs.length; i++) {
    if (specs[i] == specguige) {
      idx = i;
      break;
    }
  }
  // 拿出每一列数据
  let col = [];
  specs.forEach((_, index) => {
    col.push(Matrix[ index * specs.length + idx ]);
  });
  return col;
}
// 根据已知的sku组合来进行邻接矩阵的填充
initMatrix(skulist);
// 获取全部可用的规格属性
function getSpecOptions(params) {
  let waitchooseoption = [];
  if(params.some(Boolean)){
    // 已选属性的交集
    waitchooseoption = getIntersectionList(params.filter(Boolean))
  }else {
    // 所有属性的并集
    waitchooseoption = getUnionList(specs)
  }
  return waitchooseoption
}
// 把获取到的规格属性对应的值取出 计算
function getUnionList(specslist) {
  // let specsres = getColSum(specslist); 适用于两个分类规格的 并集
  let specsres = specslist.map((specs) => getVertexCol(specs));
  let union = [];
  specsres.forEach((item, index) => {
    // if(item && specs[index])
    if (item.some((item) => item !== 0)) {
      union.push(specs[index]);
    }
  });
  return union;
}
// 计算交集
function getIntersectionList(optionlist) {
  // let col = getColSum(optionlist) 适用于两个分类规格的
  let col = optionlist.map((item) => getVertexCol(item))  /* 用已选的每一列来 计算当前列的数据 */
  console.log(col, 'col');
  let intersectionlist = [];
  // col.forEach((item ,index) => {
  specs.forEach((item ,index) => {
    const row = col.map((i) => i[index]).filter((j) => j !== 1)
    console.log(row, 'row');
    if(isrowEqual(row)){
      intersectionlist.push(item)
    }
    // if(item >= optionlist.length && specs[index]){
    //   intersectionlist.push(specs[index])
    // }
  })
  return intersectionlist
}
// 传入一个交集行，判断内部是否互相相等
function isrowEqual(row) {
  if(row.length && row.includes(0)){
    return ;
  }
  
  let weight = -1
  // 找出权值
  if(row.length){
    row.forEach((t) => {
      if (typeof t === 'number') weight = t;
    });
    if(weight == -1){
      return isArrayUnion(row)
    } 
  }
  return row.every((item) => {
    if(typeof item == 'number'){
      return item == weight
    }
    //这一行的已选属性列间是否存在同个权值
    return item && item.includes(weight)
  })
}

// 传入多个数组，判断是否有交集
function isArrayUnion(list) {
  if(!list.length || !list[0]){
    return false
  }
  return list[0].some((i) => list.every((j) => (j && j.includes(i))))
}

const canchooseRef = computed(() => {
  return getSpecOptions(selectSpecs.value)
})

function getguige(i,j) {
  if(selectSpecs.value[i]){
      let key = selectSpecs.value[i]
      if(key == j){
        selectSpecs.value[i] = ''
        return
      }
      selectSpecs.value[i] = j
  }else {
      selectSpecs.value[i] = j
  }
}
</script>
<template>
  <div class="goodsdesc">
    <p>商品名称: {{ goodsname }}</p>
    <p>商品编号: {{ goodsid }}</p>
    <p>已选: <span v-for="value in selectSpecs">{{value}}</span></p>
    <p v-for="(i,index) in [color, type, size]">
      <p class="titlename">{{typename[index]}}</p>
      <span class="border" 
      v-for="(j, idx) in i" @click="getguige(index,j)"
      :class="[{ active: selectSpecs.includes(j)}, { disable: !canchooseRef.includes(j)}]"
      >
      {{j}}
      </span>
    </p>
  </div>
</template>
<style scoped>

 .border {
   border: 1px solid transparent;
   border-radius: 5px;
   padding: 5px 15px;
   margin: 0px 3px;
   background-color: #f5f5f5;
 }

 .active {
   border: 1px solid #2c68ff;
   color: #2c68ff;
 }

 .disable {
   border: 1px solid #f5f5f5;   
   text-decoration: line-through;
 }

 .titlename {
  font-weight: 700;
 }
</style>

