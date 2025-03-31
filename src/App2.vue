<template>
  <div class="container">
      <canvas ref="canvasRef" @mousedown="addOpData">
      </canvas>
  </div>
  <div style="position: absolute; bottom: 10px; display: flex; justify-content: center; height: 40px; width: 100vw;">
      <div style="width: 100px; height: 40px; display: flex; align-items: center; justify-content: center; color: white;"
          :style="{ backgroundColor: color }">
          <span>当前颜色</span>
      </div>
      <button style="width: 100px; height: 40px; margin-left: 10px;" @click="undo">撤销</button>
      <button style="width: 100px; height: 40px; margin-left: 10px;" @click="redo">重做</button>
      <button style="width: 100px; height: 40px; margin-left: 10px;" @click="()=>{save()}">添加保存点</button>
      <button style="width: 100px; height: 40px; margin-left: 10px;" @click="disconnect">关闭同步</button>
      <button style="width: 100px; height: 40px; margin-left: 10px;" @click="reconnect">连接同步</button>
      <button style="width: 100px; height: 40px; margin-left: 10px;" @click="clearData">清空</button>
  </div>
  <!-- 意识感知标记 -->
  <div :style="{ position: 'absolute', top: 0, left: 0, width: '100%', height: '100%', pointerEvents: 'none', overflow: 'hidden' }">
    <div v-for="(user, i) in userAwareness" :key="i" :style="{ color: user.c, border: `1px solid ${user.c}`, position: 'absolute', left: `${user.x}px`, top: `${user.y}px` }">
      {{ user.n }}
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, onMounted, Ref } from 'vue';
import * as Y from 'yjs';
import { WebsocketProvider } from 'y-websocket';
// import { WebrtcProvider } from 'y-webrtc';

const canvasRef = ref<null | HTMLCanvasElement>(null);
const ctx = ref<CanvasRenderingContext2D | null>(null);
const color = ref<string>("black");

const colors = [
  "#FF5733", "#33FF57", "#5733FF", "#FF33A2", "#A2FF33",
  "#33A2FF", "#FF33C2", "#C2FF33", "#33C2FF", "#FF3362",
  "#6233FF", "#FF336B", "#6BFF33", "#33FFA8", "#A833FF",
  "#33FFAA", "#AA33FF", "#FFAA33", "#33FF8C", "#8C33FF"
];

function getRandomColor() {
  const randomIndex = Math.floor(Math.random() * colors.length);
  return colors[randomIndex];
}

// 初始化加载 test1
const testVersionData = {
  test1: '[{"c":"#ff33c2","x":10,"y":10},{"c":"#ff33c2","x":30,"y":30},{"c":"#ff33c2","x":50,"y":50}]',
  test2: '[{"c":"#33c2ff","x":20,"y":10},{"c":"#ff33c2","x":30,"y":10},{"c":"#ff33c2","x":40,"y":10}]',
}
const testDataFromServer = {
  commitId: 'test1',
  data: testVersionData.test1,
}

/////////////////// core start ///////////////////
/**
 * http://localhost:5173/?actId=1&userId=victor
 * http://localhost:5173/?actId=1&userId=sun
 * 
 * y-websocket
 * 支持跨标签通信，同一浏览器中打开同一文档时，文档上的更改将通过跨标签通信进行交换（广播频道和本地存储作为回退）
 * 
 * 思路：
 * 1. 操作存储 yArray 中，数组天然有序，所有用户共享的记录Array完全一致
 *    大致的版本记录顺序为：[V0]-[op1]-[op2]-[v1]-[op1]-[op2]-[v2]-[op1]-[op2]
 *    第一条记录的类型 type: commit，必须保留一条版本基点用于恢复
 *    本地维护用户的所有记录，超过设置的最大版本长度后，将删除最早版本
 * 2. 并发时以ws服务器收到的先后顺序为准
 * 3. 以git版本号作为检查点，回退或冲突处理操作，需要基于检查点版本重做后续记录
 * 4. 撤销和重做的原则，参考Figma设计，应该只操作当前客户端用户的操作
 * 5. 利用localStorage设置缓存，持久化临时操作记录，避免ws服务宕机造成数据丢失
 * 6. x 可以主动选择断开同步，支持离线编辑，需要处理版本清理问题？暂时设置为不可以断开同步
 * 7. ? 断开同步的处理，按照常规单版本互斥方式 serverData/localData 处理及版本号比对
 */

// 【client信息】let baseGitCommitId = '';
const MaxVersion = 3; // 最大保留commit版本数用于回退(非op数) >=1
let localCacheOps = []; // 缓存的本地操作记录，在ws服务宕机时使用，恢复用户操作记录
let currentBaseVersionCommitId = ''; // 当前版本基点commitId
let currentOperationIndex = -1; // 当前操作索引
const urlParams = new URLSearchParams(new URL(window.location.href).search);
const actId = urlParams.get('actId') || 'demo';
const userId = urlParams.get('userId') || 'demo';
const ydoc = new Y.Doc();
// ydoc.on('beforeObserverCalls', function(tr: Y.Transaction, doc: Y.Doc) {});
// ydoc.on('beforeTransaction', function(tr: Y.Transaction, doc: Y.Doc) {});
// ydoc.on('afterTransaction', function(tr: Y.Transaction, doc: Y.Doc) {});
// ydoc.on('update', function(update: Uint8Array, origin: any, doc: Y.Doc, tr: Y.Transaction) {});
// ydoc.on('subdocs', function(changes: { loaded, added, removed }) {});
const userAwareness: Ref<any[]> = ref([]);
const clientId = ydoc.clientID;
console.log('clientId', clientId);

// 【消息类型】意识感知
type IAwarenessData = {
  n: string; // name
  c: string; // color
  x: number;
  y: number;
  s: string[]; // selectedTids
}

// 【消息类型】操作记录
type IOperateData = {
  cid?: number; // clientId
  uid?: string; // userId
  ts?: number; // version
  type: string;
  props: OpreateOpProp | OpreateCommitProp; // properties
}

type OpreateOpProp = {
  c: string;
  x: number;
  y: number;
}

type OpreateCommitProp = {
  v: string;
}

enum OpreateType {
  init = 'init',
  insert = 'insert',
  commit = 'commit',
}

/**
 * 【Y.Array】数据定义
 */
const operateData = ydoc.getArray<IOperateData>('operateData');
const initOperateDataObserve = () => {
  // 数据监听
  operateData.observe(event => {

    console.log('[observer data]', event, event.changes.delta, operateData.toArray(), JSON.stringify(operateData.toArray().map(op => {return { type: op.type, uid: op.uid }})));
    // Delta 是一种用于描述序列型数据结构的变化的格式
    // https://quilljs.com/docs/delta/
    // e.g. 初始化3个记录
    // [0] {insert: Array(3)}}
    // e.g. 插入1个记录
    // [0] {retain: 3}
    // [1] {insert: Array(1)}}
    // e.g. 撤销1次
    // [0] {retain: 2}
    // [1] {delete: 1}
    // e.g. 当retain=0时，不出现retain，如撤销唯二的两次操作
    // [0] {delete: 2}
    //
    // 包含3类操作：
    //  retain 保留
    //  insert 插入
    //  delete 删除
    // 场景分析：
    //  A [serverVersion, op1, op2, op3]
    //  B [serverVersion, op4]
    //  merge [serverVersion, op1, op2, op3]

    const deltaData = event.changes.delta;

    // 合法性校验
    if (currentOperationIndex > -1 && deltaData.length > 0) {
      const baseVersionIndex = operateData.toArray().findIndex(op => op.type === OpreateType.commit && currentBaseVersionCommitId === (op.props as OpreateCommitProp).v);
      if (baseVersionIndex < 0 || currentOperationIndex < baseVersionIndex) {
        alert('数据异常!');
        console.log(currentOperationIndex, deltaData, operateData.toArray());
        // reset();
        return;
      }
    }

    if (deltaData.length === 0) {
      // []
      // 初始化数据为空(第一个进入的用户)，自动添加版本基点作为第一个操作记录
      console.warn('[victor test] 初始化添加版本基点');
      if (!currentBaseVersionCommitId) {
        console.error('初始化数据异常');
      }
      save(currentBaseVersionCommitId);
    } else if (currentOperationIndex === -1 && deltaData.length === 1 && deltaData[0].insert) {
      // [insert] 从指定版本重做历史记录
      console.warn('[victor test] 初始化同步数据');
      const versionIndex = getIndexByCommitId(currentBaseVersionCommitId);
      if (versionIndex < 0) {
        console.exrror('版本数据异常');
        // reset();
        // window.location.reload();
        return;
      }
      // 更新操作记录索引到最近的版本记录
      currentOperationIndex = versionIndex;
      redoToOperateIndex(operateData.length - 1);
    } else if (deltaData[0]?.retain && deltaData[1]?.insert && (currentOperationIndex + 1) === deltaData[0].retain) {
      // [retain, insert] 追加更新来自其他客户端的操作
      console.warn('[victor test] 顺序追加插入记录');
      
      if (currentOperationIndex === operateData.length - 1) {
        // 只操作非当前客户端变化
        ((deltaData[0].insert || deltaData[1].insert) as Array<IOperateData>)
          .filter((data: IOperateData) => {
            return data.cid !== clientId;
          }).forEach((data: IOperateData) => {
            doHandleOperate(data);
          });
      } else {
        // 触发操作中包含当前客户端的重做
        ((deltaData[0].insert || deltaData[1].insert) as Array<IOperateData>)
          .forEach((data: IOperateData) => {
            doHandleOperate(data);
          });
        // 更新操作记录索引到最新
        currentOperationIndex = operateData.length - 1;
      }
    } else if (deltaData[0]?.insert || deltaData[1]?.insert) {
      // [insert]
      // [retain, insert]
      // 中间插入记录(重做或并发冲突记录整理)
      console.warn('[victor test] 中间位置插入记录(重做或并发冲突记录整理)');
      // 根据当前索引恢复到历史最近的版本记录
      const index = deltaData[0]?.insert ? 0 : ((deltaData[0] as any).retain - 1);
      revertToRecentVersion(index);
      // 再基于此版本记录进行回放操作
      redoToOperateIndex(operateData.length - 1);
    } else if (deltaData[0]?.delete || deltaData[1]?.delete) {
      // [delete]
      // [retain, delete]
      // 回退删除记录或清空操作
      console.warn('[victor test] 回退删除记录');
      // 根据当前索引恢复到历史最近的版本记录
      const index = deltaData[0]?.delete ? 0 : ((deltaData[0] as any).retain - 1);
      revertToRecentVersion(index);
      // 再基于此版本记录进行回放操作
      redoToOperateIndex(operateData.length - 1);
    } else {
      console.error('error');
    }

    // 缓存最新的版本号和后续操作
    const lastestVersionIndex = getRecentVersionIndex(operateData.length - 1);
    localStorage.setItem(`localData-${actId}`, JSON.stringify(operateData.slice(lastestVersionIndex)));
  });
}

/**
 * 【Y.Text】测试文件类型读写
 */
// applyDelta


/**
 * 【undoManager】
 */
const undoManager = new Y.UndoManager(operateData, {
  trackedOrigins: new Set(['insert', 'clear'])
});
const undo = () => {
  // 保留初始化记录，记录类型 type: commit，存储原始版本用于恢复
  if (operateData.length <= 1) {
    return;
  }
  undoManager.undo();
}
const redo = () => {
  undoManager.redo();
}

/**
 * 【记录操作】
 */
const addOpData = (e: any) => {
  // 如果操作栈为空，添加版本记录作为第一条记录
  if (operateData.length === 0) {
    save(currentBaseVersionCommitId);
  }
  const data: IOperateData = {
    cid: clientId,
    uid: userId,
    ts: new Date().getTime(),
    type: OpreateType.insert,
    props: { c: color.value, x: e.clientX, y: e.clientY }
  }
  // 优先本地操作，监听中过滤掉本地操作
  doHandleOperate(data);
  undoManager.stopCapturing();
  // 更新操作记录索引到最新，用于和重做区分开
  currentOperationIndex = operateData.length - 1;
  ydoc.transact(() => {
    operateData.push([data]);
  }, 'insert');
};
// 清空历史数据到堆栈第一个版本
const clearData = () => {
  // 保留第一条记录作为版本基点
  if (operateData.length === 1) {
    return;
  }
  ydoc.transact(() => {
    operateData.delete(1, operateData.length - 1);
  }, 'clear');
  // 更新操作记录索引到最新
  currentOperationIndex = 0;
  // 清空记录
  undoManager.clear();
};
// []数据异常时重置日志队列
const reset = () => {
  operateData.delete(0, operateData.length);
  currentOperationIndex = -1;
  undoManager.clear();
}

/**
 * 添加保存点[版本记录]
 * 所有用户共享的记录Array完全一致，只保留最新版本下的记录，每次保存后将历史记录存储在本地
 * 
 * MaxVersion
 */
const save = (cid?: string) => {
  let commitId = cid;
  const ts = new Date().getTime();
  // TODO 需要改为存储server返回commitId，当前默认使用时间戳作为版本号
  const v = commitId || String(ts);

  if (!commitId) {
    const baseVersionIndex = operateData.toArray().findIndex(op => op.type === OpreateType.commit && currentBaseVersionCommitId === (op.props as OpreateCommitProp).v);
    const versionDataString = generateVersionData(currentBaseVersionCommitId, operateData.slice(baseVersionIndex))!;
    // TODO 需要改为存储server返回commitId
    const commitId = saveVersionDataStr(v, versionDataString);
  }

  const data: IOperateData = {
    cid: clientId,
    uid: userId,
    ts,
    type: OpreateType.commit,
    props: { v },
  }

  undoManager.stopCapturing();
  operateData.push([data]);
  // 更新操作记录索引到最新
  currentOperationIndex = operateData.length - 1;

  // 清除超出 MaxVersion 的历史记录
  let opCommitsLength = operateData.toArray().filter(o => o.type === OpreateType.commit).length;
  while (opCommitsLength > MaxVersion) {
    const opArr = operateData.toArray();
    // 从前找到第一个版本索引，一般为 0
    let startIndex = -1;
    let endIndex = -1;
    for (let i = 0; i < opArr.length; i++) {
      if (opArr[i].type === OpreateType.commit) {
        startIndex = i;
        break;
      }
    }
    if (startIndex < 0) {
      console.error('数据异常');
      break;
    }
    for (let i = startIndex; i < opArr.length; i++) {
      if (opArr[i].type === OpreateType.commit) {
        endIndex = i;
        break;
      }
    }
    endIndex > startIndex ? (endIndex -= 1) : (endIndex = opArr.length - 1);
    // 不支持shift语法，循环内的操作会被合并
    // const ops = operateData.slice(startIndex, endIndex + 1);
    operateData.delete(startIndex, endIndex - startIndex);
    opCommitsLength -= 1;
  }
  
}

/**
 * 根据commitId版本记录索引
 */
const getIndexByCommitId = (commitId: string) => {
  let versionIndex = -1;
  // 从后向前找到最近的版本记录
  for (let i = operateData.length - 1; i >= 0; i--) {
    if (operateData.get(i).type === OpreateType.commit && (operateData.get(i).props as OpreateCommitProp).v === commitId) {
      versionIndex = i;
      break;
    }
  }
  return versionIndex;
}

/**
 * 根据当前索引找到历史最近的版本记录索引
 */
const getRecentVersionIndex = (currentIndex: number) => {
  if (currentIndex < 0) {
    return -1;
  }
  let versionIndex = -1;
  // 向前找到最近的版本记录
  for (let i = currentIndex; i >= 0; i--) {
    if (operateData.get(i).type === OpreateType.commit) {
      versionIndex = i;
      break;
    }
  }
  return versionIndex;
}

/**
 * 根据当前索引恢复到历史最近的版本记录
 */
const revertToRecentVersion = (currentIndex: number) => {
  if (currentIndex < 0) {
    return;
  }
  const versionIndex = getRecentVersionIndex(currentIndex);
  if (versionIndex < 0) {
    console.error('版本数据异常，未找到可用的版本记录');
    return false;
  }
  const commitId = (operateData.get(versionIndex).props as OpreateCommitProp).v;
  console.log(`找到最近的版本记录: ${versionIndex}=>${commitId}`);
  const loadRet = doLoadVersionByCommitId(commitId);
  if (!loadRet) {
    console.error('版本数据加载异常!');
  }
  console.log(`版本回退完成: ${currentIndex}=>${versionIndex}`);
  // 更新操作记录索引到指定索引
  currentOperationIndex = versionIndex;
}

/**
 * 回放到指定记录索引
 */
const redoToOperateIndex = (targetIndex: number, context = ctx.value) => {
  if (targetIndex <= 0) {
    return;
  }
  operateData.slice(currentOperationIndex + 1, targetIndex + 1).forEach((data: IOperateData) => {
    doHandleOperate(data, context);
  });
  console.log(`历史记录重做: ${currentOperationIndex}=>${targetIndex}`);
  // 更新操作记录索引到指定索引
  currentOperationIndex = targetIndex;
}

let reconnect = () => {};
let disconnect = () => {};
const initWebsocketServer = () => {
  /**
   * 【Provider】同步服务相关
   */
  // 创建 websocketProvider
  const websocketProvider = new WebsocketProvider(
    'ws://localhost:10086', actId, ydoc
  )
  // const webrtcProvider = new WebrtcProvider(actId, ydoc);
  // 主动断开同步
  disconnect = () => {
    websocketProvider.disconnect();
  }
  // 重连以同步
  reconnect = () => {
    websocketProvider.connect();
  }
  // 连接状态变化
  websocketProvider.on('sync', function(isSynced: boolean) {
    console.log('sync', isSynced);
  });
  // 网络连接状态变化
  websocketProvider.on('status', event => {
    console.log('status', event.status) // logs "connected" or "disconnected"
  });

  /**
   * 【Awareness】感知相关
   */
  const awareness = websocketProvider.awareness;
  console.log('当前用户列表', awareness.getStates().values());
  awareness.on('change', changes => {
    // console.log('awareness', awareness, awareness.getStates(), awareness.getStates().values());
    userAwareness.value = Array.from(awareness.getStates().values()).map(userData => userData.u).filter(userData => userData && userData.n !== userId);
  });
  document.addEventListener('mousemove', (e) => {
    // 触发感知变化(坐标)
    awareness.setLocalStateField('u', {
      n: userId,
      c: color.value,
      x: e.pageX,
      y: e.pageY,
      s: [],
    } as IAwarenessData);
  });

  (window as any).disconnect = disconnect;
  (window as any).reconnect = reconnect;
}

/////////////////// core end ///////////////////

// 根据基点版本数据和操作记录生成新的版本数据
const generateVersionData = (currentBaseVersionCommitId: string, operateData: IOperateData[]) => {
  const serverDataStr = queryVersionDataStrByCommitId(currentBaseVersionCommitId);
  if (!serverDataStr) {
    console.error('版本数据加载异常!', currentBaseVersionCommitId);
    return;
  }
  const historyData = JSON.parse(serverDataStr);
  const ops = operateData.filter(o => o.type === OpreateType.insert).map(op => op.props)
  return JSON.stringify([...historyData, ...ops]);
}

// 执行canvas点绘制
const doHandleOperate = (data: IOperateData, context = ctx.value) => {
  if (data.type === OpreateType.commit) {
    return;
  }
  console.info('draw', data);
  const props: OpreateOpProp = data.props as OpreateOpProp;
  context!.fillStyle = props.c; // 设置点的填充颜色
  context!.strokeStyle = props.c; // 设置点的边框颜色
  context!.beginPath();
  context!.moveTo(props.x, props.y);
  context!.arc(props.x, props.y, 8, 0, Math.PI * 2); // 创建一个圆形路径
  context!.fill(); // 填充路径，形成圆点
  context!.closePath();
}

// TODO 必须改为服务端获取，因为跨设备localStorage获取不到数据
// 模拟从服务器端获取和保存数据
const queryVersionDataStrByCommitId = (commitId: string) => {
  return testVersionData[commitId] || localStorage.getItem(`commitId-${commitId}`);
}
const saveVersionDataStr = (commitId: string, versionDataStr: string) => {
  localStorage.setItem(`commitId-${commitId}`, versionDataStr);
}

// 根据版本号加载内容到画布
const doLoadVersionByCommitId = (commitId, context = ctx.value) => {
  const serverDataStr = queryVersionDataStrByCommitId(commitId);
  if (!serverDataStr) {
    console.error('版本数据加载异常', commitId);
    return false;
  }
  // 清空 Canvas
  context!.clearRect(0, 0, canvasRef.value!.width, canvasRef.value!.height);
  JSON.parse(serverDataStr).forEach((item: any) => {
    doHandleOperate({
      type: OpreateType.insert,
      props: item,
    });
  });
  console.log('版本数据加载完成', commitId);
  return true;
}

// 初始化完成后设置绘画参数
onMounted(() => {
  if (canvasRef.value) {
    // 初始化后随机选择一种颜色
    color.value = getRandomColor()
    canvasRef.value.height = window.innerHeight - 10;
    canvasRef.value.width = window.innerWidth;
    const context = canvasRef.value.getContext('2d');
    if (context) {
      ctx.value = context;
      context.lineWidth = 5;
      context.fillStyle = color.value; // 设置点的填充颜色
      context.strokeStyle = color.value; // 设置点的边框颜色
      context.lineJoin = 'round';
    }

    const localCacheString = localStorage.getItem(`localData-${actId}`);
    if (localCacheString) {
      try {
        localCacheOps = JSON.parse(localCacheString);
        currentBaseVersionCommitId = (localCacheOps[0] as any).props.v;
      } catch (e) {
        console.error(e, localCacheString);
        return;
      }
    }
    if (!currentBaseVersionCommitId) {
      const serverData = JSON.parse(testDataFromServer.data);
      currentBaseVersionCommitId = testDataFromServer.commitId;
    }
    
    const loadRet = doLoadVersionByCommitId(currentBaseVersionCommitId);
    if (!loadRet) {
      console.error('版本数据加载异常!');
    }
    console.log('从服务器获取数据作为版本基点, commitId:', currentBaseVersionCommitId);

    // 启动ws
    initWebsocketServer();
    // 启动ws数据同步
    initOperateDataObserve();
  }
});
</script>

<style>
.container {
  width: 100%; height: 100%; overflow: hidden;
}
</style>