import javafx.animation.KeyFrame;
import javafx.animation.Timeline;
import javafx.application.Application;
import javafx.application.Platform;
import javafx.scene.Scene;
import javafx.scene.control.TextArea;
import javafx.scene.control.TextField;
import javafx.scene.layout.HBox;
import javafx.scene.layout.Pane;
import javafx.scene.layout.VBox;
import javafx.scene.paint.Color;
import javafx.scene.shape.Rectangle;
import javafx.scene.text.Text;
import javafx.stage.Stage;

import javafx.scene.control.Button;

import javafx.scene.control.Alert;
import javafx.scene.layout.GridPane;
import javafx.util.Duration;

import java.io.BufferedReader;
import java.io.IOException;
import java.io.InputStreamReader;
import java.io.PrintWriter;
import java.net.ServerSocket;
import java.net.Socket;
import java.util.*;
import java.util.concurrent.ConcurrentHashMap;

public class GameApp extends Application {
    private String currentPlayer = "";
    private String opponentName = "";
    private Button[][] board = new Button[3][3];
    private boolean gameEnded = false;
    private WinningStrategy winningStrategy = new DefaultWinningStrategy();
    private Map<String, String> playerAvailability = new HashMap<>();
    private Set<PrintWriter> clientWriters = ConcurrentHashMap.newKeySet();

    public GameApp() {
        playerAvailability.put("Алихан", "12:00");
        playerAvailability.put("Жұлдыз", "13:00");
        playerAvailability.put("Алмас", "13:00");
        playerAvailability.put("Еділ", "15:00");
        playerAvailability.put("Дина", "16:00");
    }

    public void start(Stage primaryStage) {
        primaryStage.setTitle("Крестики-нолики");

        GameFacade gameFacade = new GameFacade();

        Button multiplayerButton = new Button("Мультиплеер");
        Button singlePlayerButton = new Button("Одиночная игра");

        Button chatButton = new Button("Чат");
        chatButton.setOnAction(e -> gameFacade.startChatServer());


        singlePlayerButton.setOnAction(e -> gameFacade.startSnakeGame(new Stage()));
        multiplayerButton.setOnAction(e -> showTimeInputWindow(primaryStage));

        VBox layout = new VBox(10);
        layout.getChildren().addAll(multiplayerButton, singlePlayerButton);
        layout.getChildren().add(chatButton);

        Scene scene = new Scene(layout, 300, 100);
        primaryStage.setScene(scene);
        primaryStage.show();
    }
class Chat {
    private void openChatWindow() {
        Stage chatStage = new Stage();
        VBox chatLayout = new VBox(10);
        TextArea chatArea = new TextArea();
        TextField chatInput = new TextField();

        new Thread(() -> {

            try (ServerSocket serverSocket = new ServerSocket(12345)) {
                while (true) {
                    Socket clientSocket = serverSocket.accept();
                    new Thread(new ClientHandler(clientSocket, chatArea)).start();
                }
            } catch (IOException e) {
                e.printStackTrace();
            }
        }).start();

        chatInput.setOnAction(event -> {
            String message = "Ты: " + chatInput.getText();
            broadcastMessage(message);
            chatArea.appendText(message + "\n");
            chatInput.clear();
        });

        chatLayout.getChildren().addAll(chatArea, chatInput);
        chatStage.setScene(new Scene(chatLayout, 300, 200));
        chatStage.setTitle("Чат сервера");
        chatStage.show();
    }


    private void broadcastMessage(String message) {
        for (PrintWriter writer : clientWriters) {
            writer.println(message);
        }
    }

    class ClientHandler implements Runnable {
        private Socket clientSocket;
        private TextArea chatArea;
        private BufferedReader reader;
        private Set<PrintWriter> clientWriters = ConcurrentHashMap.newKeySet();

        public ClientHandler(Socket socket, TextArea chatArea) throws IOException {
            this.clientSocket = socket;
            this.chatArea = chatArea;
            PrintWriter writer = new PrintWriter(clientSocket.getOutputStream(), true);
            clientWriters.add(writer);
            reader = new BufferedReader(new InputStreamReader(socket.getInputStream()));
        }

        public void run() {
            try {
                String message;
                PrintWriter writer = new PrintWriter(clientSocket.getOutputStream(), true);
                clientWriters.add(writer);
                while ((message = reader.readLine()) != null) {
                    String finalMessage = "Соперник : " + message;
                    Platform.runLater(() -> chatArea.appendText(finalMessage + "\n"));
                }
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }

}
    private void showTimeInputWindow(Stage primaryStage) {
        Stage timeInputStage = new Stage();
        VBox timeLayout = new VBox(10);
        TextField timeInput = new TextField("Введите время (например, 15:00)");
        Button submitButton = new Button("Подтвердить");

        submitButton.setOnAction(e -> {
            String selectedTime = timeInput.getText();
            timeInputStage.close();
            showAvailablePlayers(selectedTime, primaryStage);
        });

        timeLayout.getChildren().addAll(timeInput, submitButton);
        timeInputStage.setScene(new Scene(timeLayout, 300, 100));
        timeInputStage.show();
    }

    private void showAvailablePlayers(String selectedTime, Stage primaryStage) {
        VBox layout = new VBox(10);
        boolean isAnyPlayerAvailable = false;

        for (Map.Entry<String, String> entry : playerAvailability.entrySet()) {
            if (entry.getValue().equals(selectedTime)) {
                isAnyPlayerAvailable = true;
                HBox playerBox = new HBox(10);
                Text playerName = new Text(entry.getKey());
                Button playButton = new Button("Играть");
                playButton.setOnAction(e -> {
                    opponentName = entry.getKey();
                    primaryStage.close();
                    new TicTacToeGame().startTicTacToeGame(new Stage());
                });
                playerBox.getChildren().addAll(playerName, playButton);
                layout.getChildren().add(playerBox);
            }
        }

        if (!isAnyPlayerAvailable) {
            Text noPlayersText = new Text("Никто сейчас не свободен.");
            layout.getChildren().add(noPlayersText);
        }

        primaryStage.setScene(new Scene(layout, 300, 200));
        primaryStage.show();
    }


public class TicTacToeGame{
    private void startTicTacToeGame(Stage gameStage) {
        chooseFirstPlayer();
        showFirstPlayerAlert();

        GridPane gridPane = new GridPane();

        for (int i = 0; i < 3; i++) {
            for (int j = 0; j < 3; j++) {
                Button button = ButtonFactory.createButton(100);
                int finalI = i;
                int finalJ = j;
                button.setOnAction(event -> {
                    Command makeMove = new MakeMoveCommand(button, currentPlayer);
                    makeMove.execute();
                    if (winningStrategy.checkForWin(board)) {
                        gameEnded = true;
                        showWinner(gameStage);
                    } else if (checkForDraw()) {
                        gameEnded = true;
                        showDraw(gameStage);
                    } else {
                        switchPlayer();
                    }
                });
                board[i][j] = button;
                gridPane.add(button, j, i);
            }
        }

        gameStage.setScene(new Scene(gridPane, 300, 300));
        gameStage.setTitle("Крестики-нолики");
        gameStage.show();
    }

    private void chooseFirstPlayer() {
        Random random = new Random();
        if (random.nextBoolean()) {
            currentPlayer = "X";
        } else {
            currentPlayer = "O";

        }
    }

    private void showFirstPlayerAlert() {
        Alert alert = new Alert(Alert.AlertType.INFORMATION);
        alert.setTitle("Начало игры");
        alert.setHeaderText("Первый ход");
        String firstPlayerName = currentPlayer.equals("X") ? "Вы" : opponentName;
        alert.setContentText("Первым ходит: " + firstPlayerName + " (X)");
        alert.showAndWait();
    }

    private void showWinner(Stage gameStage) {
        String winnerName = currentPlayer.equals("X") ? "Вы" : opponentName;
        Alert alert = new Alert(Alert.AlertType.INFORMATION);
        alert.setTitle("Игра окончена");
        alert.setHeaderText(null);
        alert.setContentText("Победил " + winnerName + "!");
        alert.showAndWait().ifPresent(response -> gameStage.close());
    }
    private boolean checkForDraw() {
        for (int i = 0; i < 3; i++) {
            for (int j = 0; j < 3; j++) {
                if (board[i][j].getText().isEmpty()) {
                    return false;
                }
            }
        }
        return !winningStrategy.checkForWin(board);
    }
    private void showDraw(Stage gameStage) {
        Alert alert = new Alert(Alert.AlertType.INFORMATION);
        alert.setTitle("Игра окончена");
        alert.setHeaderText(null);
        alert.setContentText("Игра окончилась вничью!");
        alert.showAndWait().ifPresent(response -> gameStage.close());
    }

    private void switchPlayer() {
        currentPlayer = currentPlayer.equals("O") ? "X" : "O";
    }

    public static void main(String[] args) {
        launch(args);
    }
}


class GameFacade {
    private TicTacToeGame ticTacToeGame;
    private SnakeGame snakeGame;

    private Chat chatServer;

    public GameFacade() {
        ticTacToeGame = new TicTacToeGame();
        snakeGame = new SnakeGame();
        chatServer = new Chat();
    }

    public void startTicTacToeGame(Stage stage) {
        ticTacToeGame.startTicTacToeGame(stage);
    }

    public void startSnakeGame(Stage stage) {
        snakeGame.start(stage);
    }
    public void startChatServer() {
        chatServer.openChatWindow();
    }

}
}
