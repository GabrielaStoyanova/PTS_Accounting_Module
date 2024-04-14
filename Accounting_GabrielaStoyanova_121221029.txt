const mysql = require('mysql2/promise');

class Database {
    constructor(config) {
        this.pool = mysql.createPool(config);
    }

    async query(sql, params = []) {
        const connection = await this.pool.getConnection();
        try {
            const [rows, fields] = await connection.query(sql, params);
            return rows;
        } finally {
            connection.release();
        }
    }
}

class Bank {
    constructor(database) {
        this.db = database;
    }

    async getBankById(bankId) {
        const banks = await this.db.query('SELECT * FROM banks WHERE id = ?', [bankId]);
        return banks[0];
    }

    async getBankByName(bankName) {
        const bank = await this.db.query('SELECT * FROM banks WHERE bank_name = ?', [bankName]);
        return bank[0];
    }
}

class UserProfile {
    constructor(database) {
        this.db = database;
        this.userType = new UserType(database);
    }

    async getUserById(userId) {
        const users = await this.db.query('SELECT * FROM user_profiles WHERE id = ?', [userId]);
        return users[0];
    }

    async getUserType(userId) {
        const userProfile = await this.getUserById(userId);
        const userType = await this.userType.getUserTypeById(userProfile.user_type_id);
        return userType.user_type;
    }

    async getUserByName(userName) {
        const user = await this.db.query('SELECT * FROM user_profiles WHERE user_name = ?', [userName]);
        return user[0];
    }
}

class UserType {
    constructor(database) {
        this.db = database;
    }

    async getUserTypeById(typeId) {
        const userTypes = await this.db.query('SELECT * FROM user_types WHERE id = ?', [typeId]);
        return userTypes[0];
    }
}

class BankAccounts {
    constructor(database) {
        this.db = database;
        this.bank = new Bank(database);
        this.userProfile = new UserProfile(database);
    }

    async getBankAccountsWithDetails() {
        const bankAccounts = await this.db.query('SELECT * FROM bank_accounts');
        const promises = bankAccounts.map(async (account) => {
            const bank = await this.bank.getBankById(account.bank_id);
            const user = await this.userProfile.getUserById(account.user_id);
            return { ...account, bank_name: bank.bank_name, user_name: user.user_name };
        });
        return Promise.all(promises);
    }

    async addBankAccount(accountNumber, bankName, userName) {
    	// Retrieve bank ID based on bank name
    	const bank = await this.bank.getBankByName(bankName);
    	if (!bank) {
        	throw new Error('Bank with this name does not exist.');
    	}
    
    	// Retrieve user ID based on user name
    	const user = await this.userProfile.getUserByName(userName);
    	if (!user) {
        	throw new Error('User with this name does not exist.');
    	}

    	// Check if the bank account already exists
    	const existingAccount = await this.db.query('SELECT * FROM bank_accounts WHERE account_number = ?', [accountNumber]);
    	if (existingAccount.length > 0) {
        	throw new Error('Bank account with this account number already exists.');
    	}

    	// Add the new bank account
    	await this.db.query('INSERT INTO bank_accounts (account_number, bank_id, user_id) VALUES (?, ?, ?)', [accountNumber, 	bank.id, user.id]);
    	return true;
    }

    async editBankAccount(accountId, accountNumber, bankName, userName) {
    	// Retrieve bank ID based on bank name
    	const bank = await this.bank.getBankByName(bankName);
    	if (!bank) {
        	throw new Error('Bank with this name does not exist.');
    	}
    
    	// Retrieve user ID based on user name
    	const user = await this.userProfile.getUserByName(userName);
    	if (!user) {
        	throw new Error('User with this name does not exist.');
    	}

    	// Check if the bank account exists
    	const existingAccount = await this.db.query('SELECT * FROM bank_accounts WHERE id = ?', [accountId]);
    	if (existingAccount.length === 0) {
        	throw new Error('Bank account with this ID does not exist.');
    	}

    	// Update the bank account details
    	await this.db.query('UPDATE bank_accounts SET account_number = ?, bank_id = ?, user_id = ? WHERE id = ?', [accountNumber, bank.id, user.id, accountId]);
    	return true;
    }

    async deleteBankAccount(accountId) {
        // Check if the bank account exists
        const existingAccount = await this.db.query('SELECT * FROM bank_accounts WHERE id = ?', [accountId]);
        if (existingAccount.length === 0) {
            throw new Error('Bank account with this ID does not exist.');
        }

        // Delete the bank account
        await this.db.query('DELETE FROM bank_accounts WHERE id = ?', [accountId]);
        return true;
    }

}

// Example configuration
const dbConfig = {
    host: 'localhost',
    user: 'gaby',
    password: '121221029',
    database: 'accounting_db'
};

// Create an instance of the Database class
const db = new Database(dbConfig);

// Create an instance of the BankAccounts class
const bankAccounts = new BankAccounts(db);

// Example usage
async function main() {
    try {
        const bankAccountsWithDetails = await bankAccounts.getBankAccountsWithDetails();
        console.log('Bank Accounts with details:', bankAccountsWithDetails);
    } catch (error) {
        console.error('Error:', error);
    }
}

main();
