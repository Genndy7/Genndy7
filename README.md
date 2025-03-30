import os
from typing import List, Dict, Optional
from dotenv import load_dotenv
from iogram import Iogram, types, html

# Load environment variables
load_dotenv()

class Book:
    """Class representing a book in the library"""
    def __init__(self, title: str, author: str, isbn: str, quantity: int = 1):
        self.title = title
        self.author = author
        self.isbn = isbn
        self.quantity = quantity
        self.borrowed_by: List[int] = []  # List of user IDs who borrowed this book

    def __str__(self) -> str:
        return f"{self.title} by {self.author} (ISBN: {self.isbn}) - Available: {self.available_copies()}"

    def available_copies(self) -> int:
        return self.quantity - len(self.borrowed_by)

    def borrow(self, user_id: int) -> bool:
        if self.available_copies() > 0 and user_id not in self.borrowed_by:
            self.borrowed_by.append(user_id)
            return True
        return False

    def return_book(self, user_id: int) -> bool:
        if user_id in self.borrowed_by:
            self.borrowed_by.remove(user_id)
            return True
        return False


class User:
    """Class representing a library user"""
    def __init__(self, user_id: int, name: str, contact: str = ""):
        self.user_id = user_id
        self.name = name
        self.contact = contact
        self.borrowed_books: List[str] = []  # List of ISBNs of borrowed books

    def __str__(self) -> str:
        return f"{self.name} (ID: {self.user_id}, Contact: {self.contact})"


class Library:
    """Class representing the library management system"""
    def __init__(self):
        self.books: Dict[str, Book] = {}  # ISBN: Book
        self.users: Dict[int, User] = {}  # user_id: User

    def add_book(self, book: Book) -> None:
        if book.isbn in self.books:
            self.books[book.isbn].quantity += book.quantity
        else:
            self.books[book.isbn] = book

    def remove_book(self, isbn: str, quantity: int = 1) -> bool:
        if isbn in self.books:
            if self.books[isbn].quantity - quantity >= 0:
                self.books[isbn].quantity -= quantity
                if self.books[isbn].quantity == 0:
                    del self.books[isbn]
                return True
        return False

    def register_user(self, user: User) -> None:
        self.users[user.user_id] = user

    def borrow_book(self, user_id: int, isbn: str) -> bool:
        if user_id in self.users and isbn in self.books:
            if self.books[isbn].borrow(user_id):
                self.users[user_id].borrowed_books.append(isbn)
                return True
        return False

    def return_book(self, user_id: int, isbn: str) -> bool:
        if user_id in self.users and isbn in self.books:
            if self.books[isbn].return_book(user_id):
                if isbn in self.users[user_id].borrowed_books:
                    self.users[user_id].borrowed_books.remove(isbn)
                return True
        return False

    def search_books(self, query: str) -> List[Book]:
        query = query.lower()
        results = []
        for book in self.books.values():
            if (query in book.title.lower() or 
                query in book.author.lower() or 
                query in book.isbn.lower()):
                results.append(book)
        return results

    def get_user_books(self, user_id: int) -> List[Book]:
        if user_id in self.users:
            return [self.books[isbn] for isbn in self.users[user_id].borrowed_books if isbn in self.books]
        return []


class LibraryBot:
    """Class representing the Telegram bot interface for the library"""
    def __init__(self, token: str):
        self.bot = Iogram(token)
        self.library = Library()
        self.setup_handlers()
        
        # Add some sample data
        self._add_sample_data()

    def _add_sample_data(self):
        """Add some sample books and users for testing"""
        self.library.add_book(Book("Python Crash Course", "Eric Matthes", "9781593279288", 3))
        self.library.add_book(Book("Clean Code", "Robert C. Martin", "9780132350884", 2))
        self.library.add_book(Book("The Pragmatic Programmer", "Andrew Hunt, David Thomas", "9780201616224", 1))
        
        self.library.register_user(User(123456, "John Doe", "john@example.com"))
        self.library.register_user(User(654321, "Jane Smith", "jane@example.com"))

    def setup_handlers(self):
        """Setup all Telegram bot handlers"""
        @self.bot.message_handler(commands=['start', 'help'])
        async def send_welcome(message: types.Message):
            welcome_text = html.bold("Welcome to LibraryBot!") + "\n\n"
            welcome_text += "Available commands:\n"
            welcome_text += "/search - Search for books\n"
            welcome_text += "/borrow - Borrow a book\n"
            welcome_text += "/return - Return a book\n"
            welcome_text += "/mybooks - List your borrowed books\n"
            welcome_text += "/register - Register as a library user\n"
            await message.reply(welcome_text, parse_mode='HTML')

        @self.bot.message_handler(commands=['register'])
        async def register_user(message: types.Message):
            user_id = message.from_user.id
            if user_id in self.library.users:
                await message.reply("You are already registered!")
                return
            
            # Ask for name and contact info
            await self.bot.send_message(
                message.chat.id,
                "Please enter your name and contact information (email/phone) separated by comma:",
                reply_markup=types.ForceReply()
            )
            
            @self.bot.message_handler(reply_to_message=message)
            async def process_registration(reply: types.Message):
                try:
                    name, contact = reply.text.split(',', 1)
                    name = name.strip()
                    contact = contact.strip()
                    
                    user = User(user_id, name, contact)
                    self.library.register_user(user)
                    await reply.reply(f"Successfully registered as {name}!")
                except ValueError:
                    await reply.reply("Invalid format. Please try again.")

        @self.bot.message_handler(commands=['search'])
        async def search_books(message: types.Message):
            await self.bot.send_message(
                message.chat.id,
                "Enter search term (title, author, or ISBN):",
                reply_markup=types.ForceReply()
            )
            
            @self.bot.message_handler(reply_to_message=message)
            async def process_search(reply: types.Message):
                query = reply.text
                results = self.library.search_books(query)
                
                if not results:
                    await reply.reply("No books found matching your search.")
                    return
                
                response = html.bold("Search Results:") + "\n\n"
                for book in results:
                    response += f"- {book}\n"
                
                await reply.reply(response, parse_mode='HTML')

        @self.bot.message_handler(commands=['borrow'])
        async def borrow_book(message: types.Message):
            user_id = message.from_user.id
            if user_id not in self.library.users:
                await message.reply("Please register first using /register")
                return
                
            await self.bot.send_message(
                message.chat.id,
                "Enter the ISBN of the book you want to borrow:",
                reply_markup=types.ForceReply()
            )
            
            @self.bot.message_handler(reply_to_message=message)
            async def process_borrow(reply: types.Message):
                isbn = reply.text.strip()
                if self.library.borrow_book(user_id, isbn):
                    book = self.library.books[isbn]
                    await reply.reply(f"Successfully borrowed '{book.title}'!")
                else:
                    await reply.reply("Could not borrow the book. It might be unavailable or the ISBN is incorrect.")

        @self.bot.message_handler(commands=['return'])
        async def return_book(message: types.Message):
            user_id = message.from_user.id
            if user_id not in self.library.users:
                await message.reply("Please register first using /register")
                return
                
            await self.bot.send_message(
                message.chat.id,
                "Enter the ISBN of the book you want to return:",
                reply_markup=types.ForceReply()
            )
            
            @self.bot.message_handler(reply_to_message=message)
            async def process_return(reply: types.Message):
                isbn = reply.text.strip()
                if self.library.return_book(user_id, isbn):
                    book = self.library.books[isbn]
                    await reply.reply(f"Successfully returned '{book.title}'!")
                else:
                    await reply.reply("Could not return the book. You might not have borrowed it or the ISBN is incorrect.")

        @self.bot.message_handler(commands=['mybooks'])
        async def list_my_books(message: types.Message):
            user_id = message.from_user.id
            if user_id not in self.library.users:
                await message.reply("Please register first using /register")
                return
                
            books = self.library.get_user_books(user_id)
            if not books:
                await message.reply("You haven't borrowed any books yet.")
                return
                
            response = html.bold("Your Borrowed Books:") + "\n\n"
            for book in books:
                response += f"- {book.title} by {book.author} (ISBN: {book.isbn})\n"
                
            await message.reply(response, parse_mode='HTML')

    def run(self):
        """Start the Telegram bot"""
        print("Library bot is running...")
        self.bot.run()


if __name__ == "__main__":
    # Get bot token from environment variable
    token = os.getenv("TELEGRAM_BOT_TOKEN")
    if not token:
        raise ValueError("Please set TELEGRAM_BOT_TOKEN in your .env file")
    
    bot = LibraryBot(token)
    bot.run()
