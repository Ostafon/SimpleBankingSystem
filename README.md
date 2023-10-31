# SimpleBankingSystem
**package banking;

import java.sql.*;
import java.util.HashMap;
import java.util.Map;
import java.util.Random;
import java.util.Scanner;

public class SimpleBankingSystem {
    private Connection connection;
    private Scanner scanner;
    private Map<String, Account> accounts;

    public SimpleBankingSystem(String dbFileName) {
        try {
            String url = "jdbc:sqlite:" + dbFileName;
            connection = DriverManager.getConnection(url);
            createTableIfNotExists();
        } catch (SQLException e) {
            System.err.println("Database connection error: " + e.getMessage());
            System.exit(1);
        }

        scanner = new Scanner(System.in);
        accounts = new HashMap<>();
    }

    public static void main(String[] args) {
        String dbFileName = "card.s3db"; // Specify the desired database file name
        SimpleBankingSystem bankingSystem = new SimpleBankingSystem(dbFileName);
        bankingSystem.run();
    }

    private void createTableIfNotExists() {
        try {
            DatabaseMetaData metaData = connection.getMetaData();
            ResultSet tables = metaData.getTables(null, null, "card", null);

            if (!tables.next()) {
                String createTableSQL = "CREATE TABLE IF NOT EXISTS card ("
                        + "id INTEGER PRIMARY KEY AUTOINCREMENT,"
                        + "number TEXT,"
                        + "pin TEXT,"
                        + "balance INTEGER DEFAULT 0"
                        + ")";
                try (Statement statement = connection.createStatement()) {
                    statement.executeUpdate(createTableSQL);
                }
                System.out.println("Table 'card' created successfully.");
            }
        } catch (SQLException e) {
            System.err.println("Error creating or checking database table: " + e.getMessage());
        }
    }

    private void run() {
        boolean running = true;
        while (running) {
            System.out.println("1. Create an account\n2. Log into an account\n0. Exit");
            int choice = scanner.nextInt();

            switch (choice) {
                case 1:
                    createAccount();
                    break;

                case 2:
                    logIntoAccount();
                    break;

                case 0:
                    System.out.println("Goodbye!");
                    running = false;
                    closeResources();
                    break;

                default:
                    break;
            }
        }

        System.exit(0); // Terminate the program
    }

    private void createAccount() {
        String cardNumber = generateCardNumber();
        String pin = generatePin();

        try {
            String insertSQL = "INSERT INTO card (number, pin) VALUES (?, ?)";
            try (PreparedStatement insertStatement = connection.prepareStatement(insertSQL)) {
                insertStatement.setString(1, cardNumber);
                insertStatement.setString(2, pin);
                int rowsAffected = insertStatement.executeUpdate();
                if (rowsAffected == 1) {
                    System.out.println("Your account has been created");
                    System.out.println("Your card number:");
                    System.out.println(cardNumber);
                    System.out.println("Your card PIN:");
                    System.out.println(pin);
                    accounts.put(cardNumber, new Account(cardNumber, pin, 0));
                } else {
                    System.out.println("Failed to create an account.");
                }
            }
        } catch (SQLException e) {
            System.err.println("Error creating account: " + e.getMessage());
        }
    }

    private void logIntoAccount() {
        System.out.println("Enter your card number:");
        String cardNumber = scanner.next();
        System.out.println("Enter your PIN:");
        String pin = scanner.next();

        try {
            String querySQL = "SELECT * FROM card WHERE number = ?";
            try (PreparedStatement queryStatement = connection.prepareStatement(querySQL)) {
                queryStatement.setString(1, cardNumber);
                try (ResultSet resultSet = queryStatement.executeQuery()) {
                    if (resultSet.next()) {
                        String storedPin = resultSet.getString("pin");
                        if (pin.equals(storedPin)) {
                            Account account = new Account(cardNumber, pin, resultSet.getInt("balance"));
                            System.out.println("You have successfully logged in!");
                            performAccountOperations(account);
                            return;
                        }
                    }
                }
            }
        } catch (SQLException e) {
            System.err.println("Error logging into account: " + e.getMessage());
        }

        System.out.println("Wrong card number or PIN!");
    }

    private void performAccountOperations(Account account) {
        while (true) {
            System.out.println("1. Balance\n2. Add income\n3. Do transfer\n4. Close account\n5. Log out\n0. Exit");
            int choice = scanner.nextInt();

            switch (choice) {
                case 1:
                    int balance = getBalance(account.getCardNumber());
                    System.out.println("Balance: " + balance);
                    break;

                case 2:
                    System.out.println("Enter income:");
                    int income = scanner.nextInt();
                    addIncome(account.getCardNumber(), income);
                    System.out.println("Income was added!");
                    break;

                case 3:
                    System.out.println("Transfer\nEnter card number:");
                    String targetCardNumber = scanner.next();

                    if (!isLuhnValid(targetCardNumber)) {
                        System.out.println("Probably you made a mistake in the card number. Please try again!");
                        break;
                    }

                    if (targetCardNumber.equals(account.getCardNumber())) {
                        System.out.println("You can't transfer money to the same account!");
                        break;
                    }

                    int targetBalance = getBalance(targetCardNumber);
                    if (targetBalance == -1) {
                        System.out.println("Such a card does not exist.");
                        break;
                    }

                    System.out.println("Enter how much money you want to transfer:");
                    int amount = scanner.nextInt();

                    String senderCardNumber = account.getCardNumber();
                    boolean success = transferMoney(senderCardNumber, targetCardNumber, amount);

                    if (success) {
                        System.out.println("Success!");
                    } else {
                        System.out.println("Error transferring money.");
                    }
                    break;

                case 4:
                    closeAccount(account.getCardNumber());
                    System.out.println("The account has been closed!");
                    return;

                case 5:
                    System.out.println("You have successfully logged out!");
                    return;

                case 0:
                    System.out.println("Goodbye!");
                    closeResources();
                    System.exit(0);
                    break;

                default:
                    break;
            }
        }
    }

    private int getBalance(String cardNumber) {
        try {
            String querySQL = "SELECT balance FROM card WHERE number = ?";
            try (PreparedStatement queryStatement = connection.prepareStatement(querySQL)) {
                queryStatement.setString(1, cardNumber);
                try (ResultSet resultSet = queryStatement.executeQuery()) {
                    if (resultSet.next()) {
                        return resultSet.getInt("balance");
                    }
                }
            }
        } catch (SQLException e) {
            System.err.println("Error getting balance: " + e.getMessage());
        }
        return -1;
    }

    private void addIncome(String cardNumber, int income) {
        try {
            String updateSQL = "UPDATE card SET balance = balance + ? WHERE number = ?";
            try (PreparedStatement updateStatement = connection.prepareStatement(updateSQL)) {
                updateStatement.setInt(1, income);
                updateStatement.setString(2, cardNumber);
                updateStatement.executeUpdate();
            }
        } catch (SQLException e) {
            System.err.println("Error updating balance: " + e.getMessage());
        }
    }

    public boolean transferMoney(String senderCardNumber, String receiverCardNumber, int amount) {
        System.out.println("Transferring " + amount + " from " + senderCardNumber + " to " + receiverCardNumber);

        int senderBalance = getBalance(senderCardNumber);
        int receiverBalance = getBalance(receiverCardNumber);

        System.out.println("Sender Balance Before: " + senderBalance);
        System.out.println("Receiver Balance Before: " + receiverBalance);

        if (senderBalance >= amount) {
            try {
                connection.setAutoCommit(false);

                String debitSQL = "UPDATE card SET balance = balance - ? WHERE number = ?";
                String creditSQL = "UPDATE card SET balance = balance + ? WHERE number = ?";

                try (PreparedStatement debitStatement = connection.prepareStatement(debitSQL);
                     PreparedStatement creditStatement = connection.prepareStatement(creditSQL)) {

                    debitStatement.setInt(1, amount);
                    debitStatement.setString(2, senderCardNumber);
                    debitStatement.executeUpdate();

                    creditStatement.setInt(1, amount);
                    creditStatement.setString(2, receiverCardNumber);
                    creditStatement.executeUpdate();

                    connection.commit();
                    connection.setAutoCommit(true);

                    addIncome(receiverCardNumber, amount); // Update receiver's balance
                    addIncome(senderCardNumber, -amount); // Update sender's balance

                    System.out.println("Sender Balance After: " + getBalance(senderCardNumber));
                    System.out.println("Receiver Balance After: " + getBalance(receiverCardNumber));

                    return true;
                }
            } catch (SQLException e) {
                try {
                    connection.rollback();
                    connection.setAutoCommit(true);
                } catch (SQLException rollbackException) {
                    System.err.println("Error rolling back transaction: " + rollbackException.getMessage());
                }
                System.err.println("Error transferring money: " + e.getMessage());
            }
        } else {
            System.out.println("Not enough money!");
        }
        return false;
    }

    private void closeAccount(String cardNumber) {
        accounts.remove(cardNumber);
        try {
            String deleteSQL = "DELETE FROM card WHERE number = ?";
            try (PreparedStatement deleteStatement = connection.prepareStatement(deleteSQL)) {
                deleteStatement.setString(1, cardNumber);
                deleteStatement.executeUpdate();
            }
        } catch (SQLException e) {
            System.err.println("Error closing account: " + e.getMessage());
        }
    }

    private void closeResources() {
        try {
            if (connection != null) {
                connection.close();
            }
        } catch (SQLException e) {
            System.err.println("Error closing database connection: " + e.getMessage());
        }
        scanner.close();
    }

    private String generatePin() {
        Random random = new Random();
        int pin = random.nextInt(10000);
        return String.format("%04d", pin);
    }

    private String generateCardNumber() {
        Random random = new Random();
        StringBuilder cardNumber = new StringBuilder("400000");
        for (int i = 0; i < 9; i++) {
            cardNumber.append(random.nextInt(10));
        }
        cardNumber.append(calculateChecksum(cardNumber.toString()));
        return cardNumber.toString();
    }

    private int calculateChecksum(String cardNumber) {
        int sum = 0;
        for (int i = 0; i < cardNumber.length(); i++) {
            int digit = Character.getNumericValue(cardNumber.charAt(i));
            if (i % 2 == 0) {
                digit *= 2;
                if (digit > 9) {
                    digit -= 9;
                }
            }
            sum += digit;
        }
        return (sum % 10 == 0) ? 0 : (10 - (sum % 10));
    }

    private boolean isLuhnValid(String cardNumber) {
        int[] digits = new int[cardNumber.length()];

        for (int i = 0; i < cardNumber.length(); i++) {
            digits[i] = Integer.parseInt(String.valueOf(cardNumber.charAt(i)));
        }

        for (int i = digits.length - 2; i >= 0; i -= 2) {
            int digit = digits[i];
            digit *= 2;
            if (digit > 9) {
                digit -= 9;
            }
            digits[i] = digit;
        }

        int sum = 0;
        for (int digit : digits) {
            sum += digit;
        }

        return sum % 10 == 0;
    }

    class Account {
        private String cardNumber;
        private String pin;
        private int balance;

        public Account(String cardNumber, String pin, int balance) {
            this.cardNumber = cardNumber;
            this.pin = pin;
            this.balance = balance;
        }

        public String getCardNumber() {
            return cardNumber;
        }

        public String getPin() {
            return pin;
        }

        public int getBalance() {
            return balance;
        }
    }
}**
