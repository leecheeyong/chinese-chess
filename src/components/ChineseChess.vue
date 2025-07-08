<script setup>
import { ref, computed, nextTick } from 'vue'

const currentPlayer = ref('red')
const selectedPosition = ref(null)
const gameBoard = ref([])
const gameHistory = ref([])
const gameStatus = ref('playing')
const possibleMoves = ref([])
const lastMove = ref(null)
const capturedPieces = ref({ red: [], black: [] })


gameBoard.value = initializeBoard()



const getPieceColor = (piece) => {
  if (!piece) return null
  return pieceDisplay[piece]?.color
}

const isInPalace = (row, col, color) => {
  if (color === 'red') {
    return row >= 7 && row <= 9 && col >= 3 && col <= 5
  } else {
    return row >= 0 && row <= 2 && col >= 3 && col <= 5
  }
}

const isOnCorrectSide = (row, color) => {
  if (color === 'red') {
    return row >= 5
  } else {
    return row <= 4
  }
}

const isPathClear = (fromRow, fromCol, toRow, toCol, board) => {
  if (fromRow === toRow) {
    const start = Math.min(fromCol, toCol)
    const end = Math.max(fromCol, toCol)
    for (let col = start + 1; col < end; col++) {
      if (board[fromRow][col]) return false
    }
  } else if (fromCol === toCol) {
    const start = Math.min(fromRow, toRow)
    const end = Math.max(fromRow, toRow)
    for (let row = start + 1; row < end; row++) {
      if (board[row][fromCol]) return false
    }
  }
  return true
}

const countPiecesBetween = (fromRow, fromCol, toRow, toCol, board) => {
  let count = 0
  if (fromRow === toRow) {
    const start = Math.min(fromCol, toCol)
    const end = Math.max(fromCol, toCol)
    for (let col = start + 1; col < end; col++) {
      if (board[fromRow][col]) count++
    }
  } else if (fromCol === toCol) {
    const start = Math.min(fromRow, toRow)
    const end = Math.max(fromRow, toRow)
    for (let row = start + 1; row < end; row++) {
      if (board[row][fromCol]) count++
    }
  }
  return count
}

const isValidMove = (piece, fromRow, fromCol, toRow, toCol, board) => {
  if (fromRow === toRow && fromCol === toCol) return false
  if (toRow < 0 || toRow >= 10 || toCol < 0 || toCol >= 9) return false
  const pieceInfo = pieceDisplay[piece]
  if (!pieceInfo) return false
  const color = pieceInfo.color
  const type = pieceInfo.type
  const rowDiff = Math.abs(toRow - fromRow)
  const colDiff = Math.abs(toCol - fromCol)
  const targetPiece = board[toRow][toCol]
  if (targetPiece && getPieceColor(targetPiece) === color) {
    return false
  }
  switch (type) {
    case 'king':
      if (!isInPalace(toRow, toCol, color)) return false
      return (rowDiff === 1 && colDiff === 0) || (rowDiff === 0 && colDiff === 1)
    case 'advisor':
      if (!isInPalace(toRow, toCol, color)) return false
      return rowDiff === 1 && colDiff === 1
    case 'elephant':
      {
        if (!isOnCorrectSide(toRow, color)) return false
        if (rowDiff !== 2 || colDiff !== 2) return false
        const blockRow = fromRow + (toRow - fromRow) / 2
        const blockCol = fromCol + (toCol - fromCol) / 2
        return !board[blockRow][blockCol]
      }
    case 'horse':
      {
        if (!((rowDiff === 2 && colDiff === 1) || (rowDiff === 1 && colDiff === 2))) return false
        let blockRow, blockCol
        if (rowDiff === 2) {
          blockRow = fromRow + (toRow > fromRow ? 1 : -1)
          blockCol = fromCol
        } else {
          blockRow = fromRow
          blockCol = fromCol + (toCol > fromCol ? 1 : -1)
        }
        return !board[blockRow][blockCol]
      }
    case 'chariot':
      if (rowDiff !== 0 && colDiff !== 0) return false
      return isPathClear(fromRow, fromCol, toRow, toCol, board)
    case 'cannon':
      if (rowDiff !== 0 && colDiff !== 0) return false
      const piecesBetween = countPiecesBetween(fromRow, fromCol, toRow, toCol, board)
      if (targetPiece) {
        return piecesBetween === 1
      } else {
        return piecesBetween === 0
      }
    case 'soldier':
      if (color === 'red') {
        if (fromRow >= 5) {
          return toRow === fromRow - 1 && colDiff === 0
        } else {
          return (toRow === fromRow - 1 && colDiff === 0) || (rowDiff === 0 && colDiff === 1)
        }
      } else {
        if (fromRow <= 4) {
          return toRow === fromRow + 1 && colDiff === 0
        } else {
          return (toRow === fromRow + 1 && colDiff === 0) || (rowDiff === 0 && colDiff === 1)
        }
      }
    default:
      return false
  }
}

const getPossibleMoves = (piece, fromRow, fromCol, board) => {
  const moves = []
  for (let row = 0; row < 10; row++) {
    for (let col = 0; col < 9; col++) {
      if (isValidMove(piece, fromRow, fromCol, row, col, board)) {
        moves.push([row, col])
      }
    }
  }
  return moves
}

const isKingInCheck = (board, color) => {
  let kingRow = -1, kingCol = -1
  const kingPiece = color === 'red' ? '帅' : '將'
  for (let row = 0; row < 10; row++) {
    for (let col = 0; col < 9; col++) {
      if (board[row][col] === kingPiece) {
        kingRow = row
        kingCol = col
        break
      }
    }
    if (kingRow !== -1) break
  }
  if (kingRow === -1) return false
  const opponentColor = color === 'red' ? 'black' : 'red'
  for (let row = 0; row < 10; row++) {
    for (let col = 0; col < 9; col++) {
      const piece = board[row][col]
      if (piece && getPieceColor(piece) === opponentColor) {
        if (isValidMove(piece, row, col, kingRow, kingCol, board)) {
          return true
        }
      }
    }
  }
  return false
}

const handleCellClick = (row, col) => {
  const piece = gameBoard.value[row][col]
  if (selectedPosition.value) {
    const [fromRow, fromCol] = selectedPosition.value
    const fromPiece = gameBoard.value[fromRow][fromCol]
    if (fromRow === row && fromCol === col) {
      selectedPosition.value = null
      possibleMoves.value = []
      return
    }
    if (isValidMove(fromPiece, fromRow, fromCol, row, col, gameBoard.value)) {
      const testBoard = gameBoard.value.map(row => [...row])
      const capturedPiece = testBoard[row][col]
      testBoard[row][col] = fromPiece
      testBoard[fromRow][fromCol] = null
      if (!isKingInCheck(testBoard, currentPlayer.value)) {
        if (capturedPiece) {
          const capturedColor = getPieceColor(capturedPiece)
          capturedPieces.value[capturedColor].push(capturedPiece)
        }
        gameBoard.value[row][col] = fromPiece
        gameBoard.value[fromRow][fromCol] = null
        gameHistory.value.push({
          from: [fromRow, fromCol],
          to: [row, col],
          piece: fromPiece,
          captured: capturedPiece,
          player: currentPlayer.value
        })
        lastMove.value = { from: [fromRow, fromCol], to: [row, col] }
        currentPlayer.value = currentPlayer.value === 'red' ? 'black' : 'red'
        selectedPosition.value = null
        possibleMoves.value = []
      } else {
        selectedPosition.value = null
        possibleMoves.value = []
      }
    } else {
      if (piece && getPieceColor(piece) === currentPlayer.value) {
        selectedPosition.value = [row, col]
        possibleMoves.value = getPossibleMoves(piece, row, col, gameBoard.value)
      } else {
        selectedPosition.value = null
        possibleMoves.value = []
      }
    }
  } else {
    if (piece && getPieceColor(piece) === currentPlayer.value) {
      selectedPosition.value = [row, col]
      possibleMoves.value = getPossibleMoves(piece, row, col, gameBoard.value)
    }
  }
}

const isCellSelected = (row, col) => {
  return selectedPosition.value && selectedPosition.value[0] === row && selectedPosition.value[1] === col
}

const isPossibleMove = (row, col) => {
  return possibleMoves.value.some(([r, c]) => r === row && c === col)
}

const isLastMove = (row, col) => {
  if (!lastMove.value) return false
  const { from, to } = lastMove.value
  return (from[0] === row && from[1] === col) || (to[0] === row && to[1] === col)
}

const getCellClass = (row, col) => {
  const piece = gameBoard.value[row][col]
  let classes = 'relative flex items-center justify-center cursor-pointer transition-all duration-200 '
  classes += 'w-8 h-8 sm:w-10 sm:h-10 md:w-12 md:h-12 lg:w-14 lg:h-14 '
  classes += 'bg-amber-50 hover:bg-amber-100 '
  classes += 'border border-amber-700 '
  if (row === 4 || row === 5) {
    classes += 'bg-blue-50 '
  }
  if (isCellSelected(row, col)) {
    classes += 'bg-blue-300 ring-2 ring-blue-500 ring-opacity-70 '
  }
  if (isPossibleMove(row, col)) {
    if (piece) {
      classes += 'bg-red-200 ring-2 ring-red-500 ring-opacity-70 '
    } else {
      classes += 'bg-green-200 '
    }
  }
  if (isLastMove(row, col)) {
    classes += 'bg-yellow-200 '
  }
  return classes
}

const getPieceClass = (piece) => {
  if (!piece) return ''
  const color = getPieceColor(piece)
  let classes = 'absolute inset-1 rounded-full flex items-center justify-center font-bold shadow-lg transition-all duration-200 hover:scale-105 select-none '
  classes += 'text-xs sm:text-sm md:text-base lg:text-lg '
  if (color === 'red') {
    classes += 'bg-gradient-to-br from-red-500 to-red-700 text-white border-2 border-red-800 '
  } else {
    classes += 'bg-gradient-to-br from-gray-700 to-gray-900 text-white border-2 border-gray-900 '
  }
  return classes
}

const resetGame = () => {
  gameBoard.value = initializeBoard()
  currentPlayer.value = 'red'
  selectedPosition.value = null
  gameHistory.value = []
  gameStatus.value = 'playing'
  possibleMoves.value = []
  lastMove.value = null
  capturedPieces.value = { red: [], black: [] }
}

const undoMove = () => {
  if (gameHistory.value.length === 0) return
  const lastMoveData = gameHistory.value.pop()
  const [fromRow, fromCol] = lastMoveData.from
  const [toRow, toCol] = lastMoveData.to
  gameBoard.value[fromRow][fromCol] = lastMoveData.piece
  gameBoard.value[toRow][toCol] = lastMoveData.captured
  if (lastMoveData.captured) {
    const capturedColor = getPieceColor(lastMoveData.captured)
    const index = capturedPieces.value[capturedColor].indexOf(lastMoveData.captured)
    if (index > -1) {
      capturedPieces.value[capturedColor].splice(index, 1)
    }
  }
  currentPlayer.value = lastMoveData.player
  selectedPosition.value = null
  possibleMoves.value = []
  lastMove.value = gameHistory.value.length > 0 ? {
    from: gameHistory.value[gameHistory.value.length - 1].from,
    to: gameHistory.value[gameHistory.value.length - 1].to
  } : null
}

const currentPlayerInCheck = computed(() => {
  return isKingInCheck(gameBoard.value, currentPlayer.value)
})
</script>

<template>
  <div class="w-full max-w-7xl mx-auto px-2 sm:px-4">
    <div class="block lg:hidden space-y-4">
      <div class="bg-white/10 backdrop-blur-sm rounded-xl p-4 border border-white/20">
       
      </div>
      <div class="bg-white/10 backdrop-blur-sm rounded-xl p-3 border border-white/20">
        <div class="flex justify-center">
          <div class="inline-block bg-amber-100 rounded-lg p-2 shadow-2xl">
            <div class="grid grid-cols-9 gap-0 border-2 border-amber-900 rounded-lg overflow-hidden">
             
            </div>
          </div>
        </div>
      </div>
      <div class="bg-white/10 backdrop-blur-sm rounded-xl p-4 border border-white/20">
        
        
        
       
      </div>
    </div>
    <div class="hidden lg:grid lg:grid-cols-4 gap-6">
      <div class="space-y-6">
        
      
        
    
        <div class="bg-white/10 backdrop-blur-sm rounded-xl p-6 border border-white/20">
          
          
        
          
          
        </div>
      </div>
      <div class="lg:col-span-2">
        <div class="bg-white/10 backdrop-blur-sm rounded-xl p-6 border border-white/20">
          <div class="flex justify-center">
            <div class="inline-block bg-amber-100 rounded-lg p-4 shadow-2xl">
              <div class="grid grid-cols-9 gap-0 border-4 border-amber-900 rounded-lg overflow-hidden">
                <template v-for="(row, rowIndex) in gameBoard" :key="rowIndex">
                  <div 
                    v-for="(cell, colIndex) in row" 
                    :key="`${rowIndex}-${colIndex}`"
                    :class="getCellClass(rowIndex, colIndex)"
                    @click="handleCellClick(rowIndex, colIndex)"
                  >
                    
                  
                    <div v-if="isPossibleMove(rowIndex, colIndex) && !cell" 
                         class="absolute inset-0 flex items-center justify-center">
                      <div class="w-3 h-3 bg-green-500 rounded-full opacity-70"></div>
                    </div>
                    <div v-if="!cell && !isPossibleMove(rowIndex, colIndex)" class="absolute inset-0 flex items-center justify-center pointer-events-none">
                      <div v-if="(rowIndex === 0 || rowIndex === 2 || rowIndex === 7 || rowIndex === 9) && (colIndex === 3 || colIndex === 5)"
                           class="absolute inset-0">
                        <svg class="w-full h-full" viewBox="0 0 56 56">
                          <line x1="0" y1="0" x2="56" y2="56" stroke="#92400e" stroke-width="1" opacity="0.3"/>
                          <line x1="56" y1="0" x2="0" y2="56" stroke="#92400e" stroke-width="1" opacity="0.3"/>
                        </svg>
                      </div>
                      <div v-if="rowIndex === 4 && colIndex === 1" class="text-xs text-blue-700 font-bold rotate-180">
                        楚河
                      </div>
                      <div v-if="rowIndex === 5 && colIndex === 7" class="text-xs text-blue-700 font-bold">
                        汉界
                      </div>
                    </div>
                  </div>
                </template>
              </div>
            </div>
          </div>
        </div>
      </div>
      <div>
        <div class="bg-white/10 backdrop-blur-sm rounded-xl p-6 border border-white/20">
          </div>
          </div>
    </div>
  </div>
</template>

<style scoped>
.grid-cols-9 {
  grid-template-columns: repeat(9, minmax(0, 1fr));
}

@keyframes pulse {
  0%, 100% {
    opacity: 1;
  }
  50% {
    opacity: 0.5;
  }
}

.animate-pulse {
  animation: pulse 2s cubic-bezier(0.4, 0, 0.6, 1) infinite;
}
</style>