Import logging
import curses
import time

# Liest Wörter aus einer Datei ein
def read_words_from_file(filename):
    try:
        with open(filename, 'r', encoding='utf-8') as file:
            return [word.strip() for line in file for word in line.split(',') if word.strip()]
    except FileNotFoundError:
        raise FileNotFoundError(f"The file {filename} was not found.")
    except Exception as e:
        raise Exception(f"An error occurred: {e}")


# Speichert die Bingo-Karte in einer Datei
def save_bingo_card(game_id, card, player):
    filename = f"bingo_{game_id}_{player}.txt"
    with open(filename, "w", encoding='utf-8') as file:
        for row in card:
            file.write(",".join(row) + "\n")

            # Hauptspielschleife
def main_game_loop(stdscr, rows, cols, game, marked, message_queues, queue, color_pair, yellow_blue, win_event, player):
    max_y, max_x = stdscr.getmaxyx()  # Maximale Abmessungen des Bildschirms
    field_width = 10  # Anpassung der Feldbreite für 7x7 Raster
    field_height = 1  # Anpassung der Feldhöhe für 7x7 Raster

    while not win_event.is_set():
        key = stdscr.getch()
        if key == ord('x'):
            queue.put((None, None))
            logging.info("Game exited")
            win_event.set()
            break
        if key == curses.KEY_MOUSE:
            _, mx, my, _, _ = curses.getmouse()
            col = (mx - 2) // (field_width + 1)
            row = (my - 2) // (field_height + 1)
            if 0 <= row < rows and 0 <= col < cols:
                if (row, col) in marked:
                    marked.remove((row, col))
                    game.unmark(row, col)
                else:
                    marked.add((row, col))
                    game.mark(row, col)
                draw_card(stdscr, game.card, marked, field_width, field_height, color_pair, False)

                queue.put((row, col))

                if game.check_bingo():
                    stdscr.addstr(2 + rows * (field_height + 1), 2, f"Player {player} won!".center((field_width + 1) * cols), yellow_blue)
                    stdscr.addstr(3, 2, "Win message sent", curses.A_BOLD | color_pair)
                    stdscr.refresh()
                    for mq in message_queues:
                        send_message(mq, f"Player {player} won!")
                        stdscr.addstr(4, 2, "Win message sent", curses.A_BOLD | color_pair)
                    logging.info(f"Player {player} won!")
                    win_event.set()
                    break

                    # Überprüft, ob ein Spieler dem Spiel beitritt
def check_player_join(game_id, player_joined):
    while not player_joined.is_set():
        game_info = load_game_info(game_id)
        if len(game_info['players']) > 0:
            player_joined.set()
        time.sleep(1)

# Fügt Protokolleinträge in die Runden-Datei ein
def insert_log_entries_in_file(log_filename, round_filename):
    with open(log_filename, 'r') as log_file:
        log_data = log_file.read()
    with open(round_filename, 'a', encoding='utf-8') as round_file:
        round_file.write('\n' + log_data)

# Ruft die Hauptfunktion auf
if _name_ == "_main_":
    mode = input("Enter 's' to start a game or 'j' to join a game: ")
    if mode == 's':
        start_game()
    elif mode == 'j':
        game_id = input("Please enter the game ID to join: ")
        join_game(game_id)
    else:
        print("Invalid mode selected.")
