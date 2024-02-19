---
layout: post
author: qbb84
tags: [Chess, Bitboards, Technical, Informative]
---

Chess, often considered the king of strategy games, has captivated minds for centuries. Behind the scenes of every formidable chess engine lies a sophisticated representation of the chessboard and its pieces. In the realm of chess engine development, one powerful technique stands out: bitboards.

In this article, I'll explore the concept of bitboards and delve into the intricacies of move generation for each chess piece. This also serves as an informative guide, and collection of ideas, incase I need to come back in the future for recollection.

<i>Note: Although I try to explain the moves concisely, to understand some of the code, you will likely need basic knowledge of bitwise operations, and/or boolean algebra. </i>

## A "bit" About Bitboards

A bitboard is a compact data structure that represents the state of a chessboard using a series of 64-bit integers. Each bit in the integer corresponds to a square on the chessboard, and its state (0 or 1) represents the presence or absence of a piece on that square. Bitboards provide an efficient way to perform bitwise operations, enabling rapid computation of various chess-related tasks.

The foundations of bitboards lies in their efficiency, especially when it comes to bitwise operations. This efficiency arises from the binary nature of computation, allowing for rapid manipulation of chess-related tasks. Bitwise AND, OR, XOR, and shifting operations become powerful tools in the hands of developers when dealing with these data structures.

<img src="/images/bitboards.png">

## Calculating Move Generation

Each chess piece can be represented by a separate bitboard, simplifying move generation. The logical operations between these bitboards efficiently compute possible moves, legal positions, and attack vectors.

The following code snippets are provided for informational purposes and to illustrate the concepts of move generation in chess. It is important to note that these code snippets may need to be adapted and integrated into a broader chess engine implementation. Consider consulting official documentation and thoroughly testing any code in a suitable development environment.

## The Pawn

#### Color Definition

- Assigns numeric values (0 for WHITE, 1 for BLACK) representing the colors of chess pieces.

#### Single Move Forward

- Calculates the target square for a single move forward based on the piece's color.
  Checks if the target square is within the chessboard boundaries and unoccupied.

#### Double Move Forward (on the first pawn move)

- Considers the piece's starting rank (second for white, seventh for black).
  If applicable, calculates the double move square and checks its validity.

#### Capture Diagonally

- Determines squares diagonally adjacent to the piece for potential captures.
  Checks if the capture squares are within the board, occupied by an opponent's piece, and not blocked by friendly pieces.

#### Return value

- Returns a long integer (bitboard) with bits set for all valid pawn moves from the given square, considering single and double moves as well as captures.

```java
    private static final int WHITE = 0;
    private static final int BLACK = 1;

    public static long generatePawnMoves(int square, int color, long occupied, long enemyPieces) {
        long moves = 0L;

        int targetSquare = square + ((color == WHITE) ? 8 : -8);
        if ((targetSquare >= 0 && targetSquare < 64) && ((occupied & (1L << targetSquare)) == 0)) {
            moves |= (1L << targetSquare);

            if (((color == WHITE) && (square / 8 == 1)) || ((color == BLACK) && (square / 8 == 6))) {
                int doubleMoveSquare = square + ((color == WHITE) ? 16 : -16);
                if ((doubleMoveSquare >= 0 && doubleMoveSquare < 64) &&
                        ((occupied & (1L << doubleMoveSquare)) == 0) &&
                        ((occupied & (1L << targetSquare)) == 0)) {
                    moves |= (1L << doubleMoveSquare);
                }
            }
        }
        int[] captureSquares = {(color == WHITE) ? square + 7 : square - 7, (color == WHITE) ? square + 9 : square - 9};
        for (int captureSquare : captureSquares) {
            if ((captureSquare >= 0 && captureSquare < 64) &&
                    ((occupied & (1L << captureSquare)) != 0) &&
                    ((1L << captureSquare) & enemyPieces) != 0) {
                moves |= (1L << captureSquare);
            }
        }
        return moves;
    }
```

## The Knight (Majestic Horse)

#### Directional Movements

- Defines possible knight moves as horizontal and vertical components.

#### Generating Moves

- Iterates through directions to calculate potential target squares.
  Sets corresponding bits in the moves bitboard for valid and unoccupied target squares.

#### Return value

- Returns a long integer (a bitboard) with bits set for all valid knight moves from the given square.

```java
 public static long generateKnightMoves(int square, long occupied) {
        long moves = 0L;
        int[] directions = {6, 10, 15, 17, -6, -10, -15, -17};

        for (int direction : directions) {
            int targetSquare = square + direction;
            if ((targetSquare >= 0 && targetSquare < 64) && ((occupied & (1L << targetSquare)) == 0)) {
                moves |= (1L << targetSquare);
            }
        }
        return moves;
    }
```

## The Bishop

### Diagonal Movement

- Represents bishop moves as diagonal paths on the chessboard.

### Generating Moves:

- Iterates through diagonal directions to calculate potential target squares.
  Sets corresponding bits in the moves bitboard for valid and unoccupied target squares.

### Return value

- Returns a long integer (bitboard) with bits set for all valid bishop moves from the given square.

```java
public static long generateBishopMoves(int square, long occupied) {
    long moves = 0L;
    int[] directions = {7, 9, -7, -9};

    for (int direction : directions) {
        int targetSquare = square + direction;
        while (targetSquare >= 0 && targetSquare < 64 && ((occupied & (1L << targetSquare)) == 0)) {
            moves |= (1L << targetSquare);
            targetSquare += direction;
        }
    }
    return moves;
}
```

## The Rook

### Orthogonal Movement

- Represents rook moves as orthogonal paths on the chessboard.

### Generating Moves

- Iterates through orthogonal directions to calculate potential target squares.
  Sets corresponding bits in the moves bitboard for valid and unoccupied target squares.

### Return Value

- Returns a long integer (bitboard) with bits set for all valid rook moves from the given square.

```java
public static long generateRookMoves(int square, long occupied) {
    long moves = 0L;
    int[] directions = {8, 1, -8, -1};

    for (int direction : directions) {
        int targetSquare = square + direction;
        while (targetSquare >= 0 && targetSquare < 64 && ((occupied & (1L << targetSquare)) == 0)) {
            moves |= (1L << targetSquare);
            targetSquare += direction;
        }
    }
    return moves;
}
```

## The Queen

### Combination of Bishop and Rook Moves

- Combines the movement patterns of bishops and rooks, allowing both diagonal and orthogonal paths.

### Generating Moves

- Iterates through combined diagonal and orthogonal directions to calculate potential target squares.
  Sets corresponding bits in the moves bitboard for valid and unoccupied target squares.

### Return Value

- Returns a long integer (bitboard) with bits set for all valid queen moves from the given square.

```java
public static long generateQueenMoves(int square, long occupied) {
    long moves = 0L;
    int[] directions = {7, 9, -7, -9, 8, 1, -8, -1};

    for (int direction : directions) {
        int targetSquare = square + direction;
        while (targetSquare >= 0 && targetSquare < 64 && ((occupied & (1L << targetSquare)) == 0)) {
            moves |= (1L << targetSquare);
            targetSquare += direction;
        }
    }
    return moves;
}
```

## The King

### Adjacent Movements

- Represents king moves as adjacent squares in all eight directions.

### Generating Moves

- Iterates through all directions to calculate potential target squares.
  Sets corresponding bits in the moves bitboard for valid and unoccupied target squares.

### Return Value

- Returns a long integer (bitboard) with bits set for all valid king moves from the given square.

```java
public static long generateKingMoves(int square, long occupied) {
    long moves = 0L;
    int[] directions = {7, 8, 9, -7, -8, -9, 1, -1};

    for (int direction : directions) {
        int targetSquare = square + direction;
        if (targetSquare >= 0 && targetSquare < 64 && ((occupied & (1L << targetSquare)) == 0)) {
            moves |= (1L << targetSquare);
        }
    }
    return moves;
}
```

## To Conclude

If you got this far, congratulations! The idea behind bitboards is relatively simple, however implementations can certainly be nuanced.

I have quite a few creative ideas that can be implemented for different variants of Chess, so this post was made minimal and concise for viewer understanding.

### Recommendations

- A video visually explaining bitboards by Holistic3D <a href="https://youtu.be/MzfQ8H16n0M?si=FEtGqBc9qxAELrOG">here</a>

- The chessprogramming.org wiki articles <a href="https://www.chessprogramming.org/Bitboards#General_Bitboard_Techniques"> here </a>

<i> If you'd like to play a chess game, contact me! </i>
