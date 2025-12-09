<script setup>
import { computed, ref, watch } from 'vue'

// 接收父组件的执棋方和重置信号
const props = defineProps({
  playerColor: {
    type: String,
    default: 'white',
  },
  resetKey: {
    type: Number,
    default: 0,
  },
})

const emit = defineEmits(['gameEnded'])

const files = ['a', 'b', 'c', 'd', 'e', 'f', 'g', 'h']
const ranks = [8, 7, 6, 5, 4, 3, 2, 1]

// 根据视角翻转文件/等级
// 视角翻转：执黑时反转文件与等级，黑棋在下
const filesForView = computed(() => (props.playerColor === 'white' ? files : [...files].reverse()))
const ranksForView = computed(() => (props.playerColor === 'white' ? ranks : [...ranks].reverse()))

// 判断当前格子是否为浅色
const isLightSquare = (row, col) => {
  return (row + col) % 2 === 0
}

// 创建开局棋子布局（带棋子类型）
// 标准开局布局（含棋子类型与符号）
const createInitialPieces = () => {
  const map = {}

  // 兵
  files.forEach((file) => {
    map[`${file}2`] = { color: 'white', symbol: '♙', type: 'pawn', hasMoved: false }
    map[`${file}7`] = { color: 'black', symbol: '♟︎', type: 'pawn', hasMoved: false }
  })

  // 后排：车 马 象 后 王 象 马 车
  const backRankOrder = ['rook', 'knight', 'bishop', 'queen', 'king', 'bishop', 'knight', 'rook']
  const symbols = {
    white: { rook: '♖', knight: '♘', bishop: '♗', queen: '♕', king: '♔' },
    black: { rook: '♜', knight: '♞', bishop: '♝', queen: '♛', king: '♚' },
  }

  backRankOrder.forEach((piece, index) => {
    const file = files[index]
    map[`${file}1`] = { color: 'white', symbol: symbols.white[piece], type: piece, hasMoved: false }
    map[`${file}8`] = { color: 'black', symbol: symbols.black[piece], type: piece, hasMoved: false }
  })

  return map
}

const pieces = ref(createInitialPieces())
const selected = ref(null) // 当前选中的格子（如 "e2"）
const turn = ref('white') // 行棋方
const winner = ref(null) // 'white' | 'black' | 'draw' | null
const statusMessage = ref('')
const lastMove = ref(null) // 记录上一步，用于吃过路兵
const promotion = ref(null) // { key, color }

const promotionSymbols = {
  white: { queen: '♕', rook: '♖', bishop: '♗', knight: '♘' },
  black: { queen: '♛', rook: '♜', bishop: '♝', knight: '♞' },
}
const promotionLabels = { queen: '后', rook: '车', bishop: '象', knight: '马' }

// 获取指定坐标的棋子
const getPiece = (file, rank, board = pieces.value) => board[`${file}${rank}`]

const fileToIndex = (file) => files.indexOf(file)

// 路径是否畅通（不含起点，含终点前一格）
// 直/斜线行走路径是否无阻挡
const isPathClear = (fromFile, fromRank, toFile, toRank, board = pieces.value) => {
  const fromX = fileToIndex(fromFile)
  const toX = fileToIndex(toFile)
  const fromY = fromRank
  const toY = toRank

  const stepX = Math.sign(toX - fromX)
  const stepY = Math.sign(toY - fromY)

  let x = fromX + stepX
  let y = fromY + stepY
  while (x !== toX || y !== toY) {
    const key = `${files[x]}${y}`
    if (board[key]) return false
    x += stepX
    y += stepY
  }
  return true
}

// 基础合法性判断（不含将军、王车易位、吃过路兵、升变），可传入特定棋盘
const isLegalMove = (fromKey, toKey, board = pieces.value) => {
  const fromPiece = board[fromKey]
  if (!fromPiece) return false
  const toPiece = board[toKey]
  if (toPiece && toPiece.color === fromPiece.color) return false

  const fromFile = fromKey[0]
  const fromRank = Number(fromKey.slice(1))
  const toFile = toKey[0]
  const toRank = Number(toKey.slice(1))

  const dx = fileToIndex(toFile) - fileToIndex(fromFile)
  const dy = toRank - fromRank
  const absDx = Math.abs(dx)
  const absDy = Math.abs(dy)

  switch (fromPiece.type) {
    case 'king': {
      // 走一步或斜一步
      if (absDx <= 1 && absDy <= 1) return true
      // 王车易位
      if (absDy === 0 && absDx === 2) {
        return canCastle(fromKey, toKey, board)
      }
      return false
    }
    case 'pawn': {
      const dir = fromPiece.color === 'white' ? 1 : -1
      const startRank = fromPiece.color === 'white' ? 2 : 7
      // 前进一步
      if (dx === 0 && dy === dir && !toPiece) return true
      // 首次前进两步，路径需畅通
      if (dx === 0 && dy === 2 * dir && fromRank === startRank && !toPiece) {
        const midRank = fromRank + dir
        if (!board[`${fromFile}${midRank}`]) return true
      }
      // 斜吃子
      if (absDx === 1 && dy === dir && toPiece && toPiece.color !== fromPiece.color) return true
      // 吃过路兵：目标格为空但符合过路兵条件
      if (absDx === 1 && dy === dir && !toPiece && canEnPassant(fromKey, toKey)) return true
      return false
    }
    case 'knight': {
      return (absDx === 1 && absDy === 2) || (absDx === 2 && absDy === 1)
    }
    case 'bishop': {
      if (absDx !== absDy) return false
      return isPathClear(fromFile, fromRank, toFile, toRank, board)
    }
    case 'rook': {
      if (fromFile !== toFile && fromRank !== toRank) return false
      return isPathClear(fromFile, fromRank, toFile, toRank, board)
    }
    case 'queen': {
      if (absDx === absDy || fromFile === toFile || fromRank === toRank) {
        return isPathClear(fromFile, fromRank, toFile, toRank, board)
      }
      return false
    }
    default:
      return false
  }
}

// 执行走子/吃子
const movePiece = (fromKey, toKey) => {
  const next = { ...pieces.value }
  const movingPiece = { ...next[fromKey], hasMoved: true }

  // 处理王车易位：王水平移动两格
  if (movingPiece.type === 'king' && Math.abs(fileToIndex(toKey[0]) - fileToIndex(fromKey[0])) === 2) {
    const rank = fromKey.slice(1)
    const isKingSide = fileToIndex(toKey[0]) > fileToIndex(fromKey[0])
    const rookFrom = isKingSide ? `h${rank}` : `a${rank}`
    const rookTo = isKingSide ? `f${rank}` : `d${rank}`
    const rookPiece = { ...next[rookFrom], hasMoved: true }
    delete next[rookFrom]
    next[rookTo] = rookPiece
  }

  // 处理吃过路兵：斜走到空格，吃掉旁边对方兵
  if (
    movingPiece.type === 'pawn' &&
    !next[toKey] &&
    Math.abs(fileToIndex(toKey[0]) - fileToIndex(fromKey[0])) === 1
  ) {
    const fromRank = Number(fromKey.slice(1))
    const dir = movingPiece.color === 'white' ? 1 : -1
    const capturedKey = `${toKey[0]}${fromRank}`
    if (next[capturedKey]) {
      delete next[capturedKey]
    }
  }

  // 兵升变：抵达底线自动升为后
  if (movingPiece.type === 'pawn') {
    const toRank = Number(toKey.slice(1))
    const promotionRank = movingPiece.color === 'white' ? 8 : 1
    if (toRank === promotionRank) {
      movingPiece.type = 'queen'
      movingPiece.symbol = movingPiece.color === 'white' ? '♕' : '♛'
      promotion.value = { key: toKey, color: movingPiece.color }
    }
  }

  next[toKey] = movingPiece
  delete next[fromKey]
  pieces.value = next
}

// 应用升变选择
const applyPromotion = (pieceType) => {
  if (!promotion.value) return
  const { key, color } = promotion.value
  const current = pieces.value[key]
  if (!current || current.type !== 'queen' || current.color !== color) {
    promotion.value = null
    return
  }

  const updated = {
    ...current,
    type: pieceType,
    symbol: promotionSymbols[color][pieceType],
  }

  pieces.value = {
    ...pieces.value,
    [key]: updated,
  }

  promotion.value = null
}

// 找到指定颜色的王位置
const findKing = (color, board = pieces.value) => {
  return Object.entries(board).find(([, p]) => p.type === 'king' && p.color === color)?.[0] || null
}

// 判断一个格子是否被指定颜色攻击
const isSquareAttacked = (targetKey, byColor, board = pieces.value) => {
  for (const [fromKey, piece] of Object.entries(board)) {
    if (piece.color !== byColor) continue
    if (isLegalMove(fromKey, targetKey, board)) {
      return true
    }
  }
  return false
}

// 是否被将军
const isKingInCheck = (color, board = pieces.value) => {
  const kingPos = findKing(color, board)
  if (!kingPos) return true
  const opponent = color === 'white' ? 'black' : 'white'
  return isSquareAttacked(kingPos, opponent, board)
}

// 吃过路兵合法性判断
const canEnPassant = (fromKey, toKey) => {
  const last = lastMove.value
  if (!last) return false

  const fromPiece = pieces.value[fromKey]
  if (!fromPiece || fromPiece.type !== 'pawn') return false

  const fromFile = fromKey[0]
  const fromRank = Number(fromKey.slice(1))
  const toFile = toKey[0]
  const toRank = Number(toKey.slice(1))

  // 目标格必须为空
  if (pieces.value[toKey]) return false

  const dir = fromPiece.color === 'white' ? 1 : -1
  // 斜向一格前进
  if (Math.abs(fileToIndex(toFile) - fileToIndex(fromFile)) !== 1) return false
  if (toRank - fromRank !== dir) return false

  // 上一步必须是对方兵从初始位走两格，且停在并排格
  const lastPiece = last.piece
  if (!lastPiece || lastPiece.type !== 'pawn' || lastPiece.color === fromPiece.color) return false

  const lastFromRank = Number(last.from.slice(1))
  const lastToRank = Number(last.to.slice(1))
  const lastFromFile = last.from[0]
  const lastToFile = last.to[0]
  const oppStartRank = lastPiece.color === 'white' ? 2 : 7

  // 必须是两格直进
  if (lastFromFile !== lastToFile) return false
  if (Math.abs(lastToRank - lastFromRank) !== 2) return false
  if (lastFromRank !== oppStartRank) return false

  // 目标的文件与对方兵落点文件相同，且目标排是对方兵落点所在排的前进一步
  if (toFile !== lastToFile) return false
  if (toRank !== lastToRank + dir) return false

  return true
}

// 判断王车易位合法性
const canCastle = (fromKey, toKey, board = pieces.value) => {
  const king = board[fromKey]
  if (!king || king.type !== 'king' || king.hasMoved) return false

  const fromFile = fromKey[0]
  const fromRank = Number(fromKey.slice(1))
  const toFile = toKey[0]
  const toRank = Number(toKey.slice(1))

  // 仅允许同排水平移动两格
  if (fromRank !== toRank) return false
  const dx = fileToIndex(toFile) - fileToIndex(fromFile)
  if (Math.abs(dx) !== 2) return false

  const isKingSide = dx > 0
  const rookFile = isKingSide ? 'h' : 'a'
  const rookKey = `${rookFile}${fromRank}`
  const rook = board[rookKey]
  if (!rook || rook.type !== 'rook' || rook.color !== king.color || rook.hasMoved) return false

  // 路径必须无子（王与车之间）
  const betweenFiles = isKingSide ? ['f', 'g'] : ['b', 'c', 'd']
  for (const f of betweenFiles) {
    if (board[`${f}${fromRank}`]) return false
  }

  // 王当前位置、经过格、目标格不得被攻击
  const opponent = king.color === 'white' ? 'black' : 'white'
  const kingPath = isKingSide ? ['e', 'f', 'g'] : ['e', 'd', 'c']
  for (const f of kingPath) {
    const square = `${f}${fromRank}`
    if (isSquareAttacked(square, opponent, board)) return false
  }

  return true
}

// 判断某方是否有合法着法（不让己王被将军）
const hasAnyLegalMove = (color, board = pieces.value) => {
  for (const [fromKey, piece] of Object.entries(board)) {
    if (piece.color !== color) continue
    for (const file of files) {
      for (const rank of ranks) {
        const toKey = `${file}${rank}`
        if (!isLegalMove(fromKey, toKey, board)) continue

        const boardCopy = { ...board }
        boardCopy[toKey] = boardCopy[fromKey]
        delete boardCopy[fromKey]

        const kingPos = findKing(color, boardCopy)
        if (!kingPos) continue
        const opponent = color === 'white' ? 'black' : 'white'
        if (!isSquareAttacked(kingPos, opponent, boardCopy)) {
          return true
        }
      }
    }
  }
  return false
}

// 判定胜负/和棋
const evaluateResult = (sideToMove, moverColor) => {
  const board = pieces.value
  const kingPos = findKing(sideToMove, board)
  if (!kingPos) {
    winner.value = moverColor
    statusMessage.value = `${moverColor === 'white' ? '白方' : '黑方'}胜（对方国王被吃）`
    emit('gameEnded', winner.value)
    return
  }

  const inCheck = isKingInCheck(sideToMove, board)
  const hasMove = hasAnyLegalMove(sideToMove, board)

  if (!hasMove && inCheck) {
    winner.value = moverColor
    statusMessage.value = `${moverColor === 'white' ? '白方' : '黑方'}胜（将死）`
    emit('gameEnded', winner.value)
  } else if (!hasMove && !inCheck) {
    winner.value = 'draw'
    statusMessage.value = '和棋（无合法着）'
    emit('gameEnded', winner.value)
  } else {
    statusMessage.value = inCheck ? `${sideToMove === 'white' ? '白方' : '黑方'}被将军` : ''
  }
}

// 点击格子处理走棋
const handleSquareClick = (file, rank) => {
  const key = `${file}${rank}`
  const piece = pieces.value[key]

  // 如果未选中，则只能选中当前行棋方的棋子
  if (!selected.value) {
    if (piece && piece.color === turn.value) {
      selected.value = key
    }
    return
  }

  // 如果点击同色棋子，则切换选中
  const fromKey = selected.value
  const fromPiece = pieces.value[fromKey]
  if (piece && piece.color === fromPiece.color) {
    selected.value = key
    return
  }

  // 合法性校验
  const legal = isLegalMove(fromKey, key)
  if (!legal) return

  // 执行走子/吃子：将 from 移到 to
  movePiece(fromKey, key)

  // 轮换行棋方，并清除选中
  const mover = turn.value
  turn.value = turn.value === 'white' ? 'black' : 'white'
  selected.value = null

  // 记录本次走子（用于吃过路兵）
  lastMove.value = { from: fromKey, to: key, piece: { ...fromPiece } }

  // 胜负判定：检查下一个行棋方是否被将死/无合法着
  evaluateResult(turn.value, mover)
}

// 重置棋盘
// 重置棋盘
const resetBoard = () => {
  pieces.value = createInitialPieces()
  selected.value = null
  turn.value = 'white'
  winner.value = null
  statusMessage.value = ''
  lastMove.value = null
}

// 监听父组件的 resetKey 变化进行重置
watch(
  () => props.resetKey,
  () => resetBoard()
)
</script>

<template>
  <div class="chess-board">
    <div v-if="promotion" class="promotion-modal">
      <div class="promotion-card">
        <div class="promotion-title">选择升变棋子</div>
        <div class="promotion-options">
          <button
            v-for="type in ['queen', 'rook', 'bishop', 'knight']"
            :key="type"
            type="button"
            class="promotion-option"
            @click.stop="applyPromotion(type)"
          >
            <span class="piece" :class="promotion.color">{{ promotionSymbols[promotion.color][type] }}</span>
            <span class="promotion-label">{{ promotionLabels[type] }}</span>
          </button>
        </div>
      </div>
    </div>

    <div v-if="statusMessage || winner" class="status-bar" :class="winner ? 'finished' : 'warning'">
      <span>{{ statusMessage || (winner === 'draw' ? '和棋' : `${winner === 'white' ? '白方' : '黑方'}胜`) }}</span>
    </div>

    <!-- 顶部文件标记 (a-h) -->
    <div class="file-labels top">
      <div class="file-label" v-for="file in filesForView" :key="file">{{ file }}</div>
    </div>
    
    <div class="board-content">
      <!-- 左侧等级标记 (8-1) -->
      <div class="rank-labels left">
        <div class="rank-label" v-for="rank in ranksForView" :key="rank">{{ rank }}</div>
      </div>
      
      <!-- 棋盘格子 -->
      <div class="board-grid">
        <div
          v-for="(rank, rowIndex) in ranksForView"
          :key="`rank-${rank}`"
          class="board-row"
        >
          <div
            v-for="(file, colIndex) in filesForView"
            :key="`${file}${rank}`"
            :class="[
              'square',
              isLightSquare(rowIndex, colIndex) ? 'light' : 'dark',
              selected === `${file}${rank}` ? 'selected' : ''
            ]"
            @click="handleSquareClick(file, rank)"
          >
            <div
              v-if="getPiece(file, rank)"
              class="piece"
              :class="getPiece(file, rank)?.color"
            >
              {{ getPiece(file, rank)?.symbol }}
            </div>
          </div>
        </div>
      </div>
      
      <!-- 右侧等级标记 (8-1) -->
      <div class="rank-labels right">
        <div class="rank-label" v-for="rank in ranksForView" :key="rank">{{ rank }}</div>
      </div>
    </div>
    
    <!-- 底部文件标记 (a-h) -->
    <div class="file-labels bottom">
      <div class="file-label" v-for="file in filesForView" :key="file">{{ file }}</div>
    </div>
  </div>
</template>

