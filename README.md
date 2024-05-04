# Hexapawn
import enum
from collections import namedtuple

__all__ = [
    'Player',
    'Point',
]


class Player(enum.Enum):
    x = 1
    o = 2

    @property
    def other(self):
        return Player.x if self == Player.o else Player.o


class Point(namedtuple('Point', 'row col')):
    def __deepcopy__(self, memodict={}):
        # These are very immutable.
        return self

import copy

#from hextypes import Player, Point

__all__ = [
    'Board',
    'GameState',
    'Move',
]


class IllegalMoveError(Exception):
    pass


BOARD_SIZE = 3
ROWS = tuple(range(1, BOARD_SIZE + 1))
COLS = tuple(range(1, BOARD_SIZE + 1))
# Top left to lower right diagonal
DIAG_1 = (Point(1, 1), Point(2, 2), Point(3, 3))
# Top right to lower left diagonal
DIAG_2 = (Point(1, 3), Point(2, 2), Point(3, 1))


class Board:
    def __init__(self):
        self._grid = {}
        self._grid[Point(1, 1)] = Player.o
        self._grid[Point(1, 2)] = Player.o
        self._grid[Point(1, 3)] = Player.o
        self._grid[Point(3, 1)] = Player.x
        self._grid[Point(3, 2)] = Player.x
        self._grid[Point(3, 3)] = Player.x

    def place(self, player, point_a, point_b):
        assert self.is_on_grid(point_a)
        assert self.is_on_grid(point_b)
        #assert self._grid.get(point_b) is None
        self._grid[point_a] = None
        self._grid[point_b] = player

    @staticmethod
    def is_on_grid(point):
        return 1 <= point.row <= BOARD_SIZE and \
            1 <= point.col <= BOARD_SIZE

    def get(self, point):
        """Return the content of a point on the board.

        Returns None if the point is empty, or a Player if there is a
        stone on that point.
        """
        return self._grid.get(point)


class Move:
    def __init__(self, point_a, point_b):
        self.point_a = point_a
        self.point_b = point_b


class GameState:
    def __init__(self, board, next_player, move):
        self.board = board
        self.next_player = next_player
        self.last_move = move
        self.count = 0

    def apply_move(self, move):
        """Return the new GameState after applying the move."""
        next_board = copy.deepcopy(self.board)
        if move is not None:
            next_board.place(self.next_player, move.point_a, move.point_b)
        return GameState(next_board, self.next_player.other, move)

    @classmethod
    def new_game(cls, player):
        board = Board()
        return GameState(board, player, None)

    def is_valid_move(self, move):
        self.debug('move=' + str(move.point_a) + ' ' + str(move.point_b))

        # points must be in the grid
        if not Board.is_on_grid(move.point_a) or not Board.is_on_grid(move.point_b):
            self.debug('not Board.is_on_grid(move.point_a) or not Board.is_on_grid(move.point_b)')
            return False

        # points A and B must be different
        if move.point_a == move.point_b:
            self.debug('move.point_a == move.point_b')
            return False

        # game must be still on
        if self.is_over():
            self.debug('is_over')
            return False

        # point A must belong to the next_player
        if self.next_player != self.board.get(move.point_a):
            self.debug('self.next_player != self.board.get(move.point_a)')
            return False

        # if point b is in the same col, it must be available and cannot be 2 spots away
        if move.point_a.col == move.point_b.col and \
                self.board.get(move.point_b) is None and \
                abs(move.point_a.row - move.point_b.row) == 1:
            self.debug('if point b is in the same col, it must be available and cannot be 2 spots away')
            return True

        # if A and B belong to different players, they must be in different cols and rows
        if self.board.get(move.point_b) is not None and \
                self.board.get(move.point_a) != self.board.get(move.point_b) and \
                move.point_a.col != move.point_b.col and move.point_a.row != move.point_b.row and \
                abs(move.point_a.col - move.point_b.col) == 1 and abs(move.point_a.row - move.point_b.row) == 1:
            self.debug('if A and B belong to different players, they must be in different cols and rows')
            return True

        return False

    def debug(self, message):
        debug_flag = False
        if debug_flag:
            print(message)

    def legal_moves(self):
        self.debug('legal_moves')
        moves = []
        for row in ROWS:
            for col in COLS:
                if self.board.get(Point(row, col)) == Player.o:
                    move = Move(Point(row, col), Point(row+1, col))
                    if self.is_valid_move(move):
                        self.debug('bot - append move.point_b=' + str(move.point_b))
                        moves.append(move)
                    move = Move(Point(row, col), Point(row+1, col-1))
                    if self.is_valid_move(move):
                        self.debug('bot - append move.point_b=' + str(move.point_b))
                        moves.append(move)
                    move = Move(Point(row, col), Point(row+1, col+1))
                    if self.is_valid_move(move):
                        self.debug('bot - append move.point_b=' + str(move.point_b))
                        moves.append(move)
                else:
                    move = Move(Point(row, col), Point(row - 1, col))
                    if self.is_valid_move(move):
                        self.debug('human - append move.point_b=' + str(move.point_b))
                        moves.append(move)
                    move = Move(Point(row, col), Point(row - 1, col - 1))
                    if self.is_valid_move(move):
                        self.debug('human - append move.point_b=' + str(move.point_b))
                        moves.append(move)
                    move = Move(Point(row, col), Point(row - 1, col + 1))
                    if self.is_valid_move(move):
                        self.debug('human - append move.point_b=' + str(move.point_b))
                        moves.append(move)

        self.debug(f'len(moves)={len(moves)}')
        return moves

    def is_over(self):
        if self._has_crossed_board(Player.x):
            return True
        if self._has_crossed_board(Player.o):
            return True
        for row in ROWS:
            for col in COLS:
                if self.next_player == Player.o and self.board.get(Point(row, col)) == Player.o:
                    if self.board.get(Point(row+1, col)) is None \
                            or self.board.get(Point(row+1, col+1)) == Player.x \
                            or self.board.get(Point(row+1, col-1)) == Player.x:
                        return False
                if self.next_player == Player.x and self.board.get(Point(row, col)) == Player.x:
                    if self.board.get(Point(row - 1, col)) is None \
                            or self.board.get(Point(row - 1, col + 1)) == Player.o \
                            or self.board.get(Point(row - 1, col - 1)) == Player.o:
                        return False
        return True

    def _has_crossed_board(self, player):
        if Player.x == player:
            for col in COLS:
                if self.board.get(Point(1, col)) == Player.x:
                    return True
        else:
            for col in COLS:
                if self.board.get(Point(3, col)) == Player.o:
                    return True
        return False

    def winner(self):
        if self._has_crossed_board(Player.x):
            return Player.x
        elif self._has_crossed_board(Player.o):
            return Player.o
        elif self.next_player == Player.x:
            return Player.o
        else:
            return Player.x

from six.moves import input
from dlgo import minimax

COL_NAMES = 'ABC'


def print_board(board):
    print('   A   B   C')
    for row in (1, 2, 3):
        pieces = []
        for col in (1, 2, 3):
            piece = board.get(Point(row, col))
            if piece == Player.x:
                pieces.append('X')
            elif piece == Player.o:
                pieces.append('O')
            else:
                pieces.append(' ')
        print('%d  %s' % (row, ' | '.join(pieces)))


def point_from_coords(text):
    col_name_a = text[0]
    row_a = int(text[1])
    col_name_b = text[2]
    row_b = int(text[3])
    return Point(row_a, COL_NAMES.index(col_name_a) + 1), Point(row_b, COL_NAMES.index(col_name_b) + 1)

def playhex(first_player_bot=False):
    human_player = Player.x
    bot = minimax.MinimaxAgent()
    
    count = 0
    while True:
        if count%2 == 0:
            player = Player.o
        else:
            player = Player.x
        
        count+=1
        game = GameState.new_game(player)

#         if first_player_bot:
#             move = bot.select_move(game)
#             if move is None:
#                 print('Bot should have a move :-(')
#             game = game.apply_move(move)

        while not game.is_over():
            print_board(game.board)
            if game.next_player == human_player:
                human_move = input('-- ')
                point_a, point_b = point_from_coords(human_move.upper().strip())
                move = Move(point_a, point_b)
                if not game.is_valid_move(move):
                    print('Not a valid move')
                    continue
            else:
                move = bot.select_move(game)
                if move is None:
                    print('Bot should have a move :-(')
            game = game.apply_move(move)

        print_board(game.board)
        winner = game.winner()
        print('Winner: ' + str(winner))

        first_player_bot = not first_player_bot
        replay = input("Do you want to play again? (yes/no): ").strip().lower()
        if replay != "yes":
            break

playhex()
