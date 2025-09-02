

// Main App component
export default function App() {
  const [board, setBoard] = useState([]);
  const [currentPlayer, setCurrentPlayer] = useState('Red');
  const [winner, setWinner] = useState(null);
  const [message, setMessage] = useState('');
  const [isThinking, setIsThinking] = useState(false);
  const rows = 6;
  const cols = 7;

  // Function to initialize the board
  const initializeBoard = useCallback(() => {
    const newBoard = Array(rows).fill(null).map(() => Array(cols).fill(null));
    setBoard(newBoard);
    setCurrentPlayer('Red');
    setWinner(null);
    setIsThinking(false);
    setMessage(`Red's turn`);
  }, [rows, cols]);

  useEffect(() => {
    initializeBoard();
  }, [initializeBoard]);

  // Function to check for a winner
  const checkWinner = (currentBoard) => {
    const directions = [
      [0, 1], [1, 0], [1, 1], [1, -1]
    ];

    for (let r = 0; r < rows; r++) {
      for (let c = 0; c < cols; c++) {
        const player = currentBoard[r][c];
        if (player) {
          for (const [dr, dc] of directions) {
            let count = 0;
            for (let i = 0; i < 4; i++) {
              const newR = r + i * dr;
              const newC = c + i * dc;
              if (
                newR >= 0 && newR < rows &&
                newC >= 0 && newC < cols &&
                currentBoard[newR][newC] === player
              ) {
                count++;
              } else {
                break;
              }
            }
            if (count === 4) {
              return player;
            }
          }
        }
      }
    }
    return null;
  };

  // Function to handle a column click
  const handleColumnClick = (colIndex) => {
    if (winner || isThinking) return;

    const newBoard = board.map(row => [...row]);
    let rowIndex = -1;
    for (let i = rows - 1; i >= 0; i--) {
      if (newBoard[i][colIndex] === null) {
        newBoard[i][colIndex] = currentPlayer;
        rowIndex = i;
        break;
      }
    }

    if (rowIndex === -1) {
      setMessage("Column is full!");
      return;
    }

    setBoard(newBoard);
    const newWinner = checkWinner(newBoard);
    if (newWinner) {
      setWinner(newWinner);
      setMessage(`${newWinner} wins!`);
    } else {
      const isDraw = newBoard.every(row => row.every(cell => cell !== null));
      if (isDraw) {
        setWinner('Draw');
        setMessage("It's a draw!");
      } else {
        const nextPlayer = currentPlayer === 'Red' ? 'Yellow' : 'Red';
        setCurrentPlayer(nextPlayer);
        setMessage(`${nextPlayer}'s turn`);
      }
    }
  };

  // Function to get a strategy hint from the Gemini API
  const getStrategyHint = async () => {
    setIsThinking(true);
    const apiKey = "";
    const apiUrl = `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash-preview-05-20:generateContent?key=${apiKey}`;

    // Convert the board to a readable format for the LLM
    const boardString = board.map(row => 
      row.map(cell => cell === 'Red' ? 'R' : cell === 'Yellow' ? 'Y' : '-').join('')
    ).join('\n');

    const prompt = `You are an expert Connect Four player and strategist. Analyze the following game board and provide a concise strategic tip for the current player. The board is represented by 'R' for Red, 'Y' for Yellow, and '-' for an empty space. The board is ${rows} rows by ${cols} columns.

Current Player: ${currentPlayer}
Current Board State:
${boardString}

Provide a short, specific hint, focusing on what the current player should do to either block the opponent or set up their own win. Do not play the move yourself, just provide the hint.`;

    const payload = {
      contents: [{ parts: [{ text: prompt }] }],
    };

    try {
      const response = await fetch(apiUrl, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(payload)
      });
      const result = await response.json();
      const hint = result?.candidates?.[0]?.content?.parts?.[0]?.text || "Couldn't get a hint. Try again!";
      setMessage(`Hint: ${hint}`);
    } catch (error) {
      console.error('Error fetching hint:', error);
      setMessage("Failed to get a hint. Check the console for details.");
    } finally {
      setIsThinking(false);
    }
  };

  // Cell component to display a single cell
  const Cell = ({ value, onClick }) => {
    const colorClass = value ? `bg-${value.toLowerCase()}-500` : 'bg-white';
    return (
      <div
        className={`w-12 h-12 md:w-16 md:h-16 rounded-full border-2 border-gray-400 m-1 flex items-center justify-center cursor-pointer transition-transform duration-300 hover:scale-105 ${colorClass}`}
        onClick={onClick}
      >
        <div className={`w-10 h-10 md:w-14 md:h-14 rounded-full transition-all duration-500 transform scale-0 ${value ? 'scale-100' : ''}`} />
      </div>
    );
  };

  // Column component to handle click events
  const Column = ({ colIndex, children }) => {
    return (
      <div onClick={() => handleColumnClick(colIndex)} className="flex flex-col-reverse items-center">
        {children}
      </div>
    );
  };

  return (
    <div className="min-h-screen bg-gray-900 flex flex-col items-center justify-center p-4 text-white font-inter">
      <style>{`
        @import url('https://fonts.googleapis.com/css2?family=Inter:wght@400;700&display=swap');
        body {
          font-family: 'Inter', sans-serif;
        }
      `}</style>
      <div className="bg-gray-800 p-8 rounded-xl shadow-lg border-2 border-gray-700 w-full max-w-2xl text-center">
        <h1 className="text-4xl md:text-5xl font-bold mb-4">Connect Four</h1>
        <p className="text-xl md:text-2xl font-bold mb-6">
          {isThinking ? "Thinking..." : message}
        </p>
        <div className="grid gap-2 mb-6" style={{ gridTemplateColumns: `repeat(${cols}, 1fr)` }}>
          {Array.from({ length: cols }).map((_, colIndex) => (
            <Column key={colIndex} colIndex={colIndex}>
              {Array.from({ length: rows }).map((_, rowIndex) => (
                <Cell key={rowIndex} value={board[rowIndex] && board[rowIndex][colIndex]} />
              ))}
            </Column>
          ))}
        </div>
        <div className="flex flex-col sm:flex-row justify-center space-y-4 sm:space-y-0 sm:space-x-4">
          <button
            onClick={initializeBoard}
            className="bg-blue-600 hover:bg-blue-700 text-white font-bold py-3 px-6 rounded-full shadow-lg transition-transform duration-300 transform hover:scale-105 active:scale-95"
          >
            Reset Game
          </button>
          <button
            onClick={getStrategyHint}
            disabled={isThinking || winner}
            className="bg-purple-600 hover:bg-purple-700 text-white font-bold py-3 px-6 rounded-full shadow-lg transition-transform duration-300 transform hover:scale-105 active:scale-95 disabled:opacity-50 disabled:cursor-not-allowed"
          >
            Get Strategy Hint ✨
          </button>
        </div>
      </div>
    </div>
  );
}
