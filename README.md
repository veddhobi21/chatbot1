# chatbot1
"A Streamlit-based AI chatbot for real-time Q&amp;A, PDF processing, and secure user authentication, using Python and SQLite."

import streamlit as st
from groq import Groq
from dotenv import load_dotenv
load_dotenv()
import os
import sqlite3

# Connect to database (or create it)
conn = sqlite3.connect('users.db')
c = conn.cursor()

# Create a table for user registration
c.execute('''
    CREATE TABLE IF NOT EXISTS users (
        username TEXT PRIMARY KEY,
        password TEXT
    )
''')

conn.commit()
conn.close()
import hashlib


def register_user(username, password):
    conn = sqlite3.connect('users.db')
    c = conn.cursor()

    # Hash the password
    password_hash = hashlib.sha256(password.encode()).hexdigest()

    # Insert the user into the database
    try:
        c.execute('INSERT INTO users (username, password) VALUES (?, ?)', (username, password_hash))
        conn.commit()
        conn.close()
        return "Registration successful!"
    except sqlite3.IntegrityError:
        return "Username already exists. Please choose a different one."


def login_user(username, password):
    conn = sqlite3.connect('users.db')
    c = conn.cursor()

    # Hash the entered password
    password_hash = hashlib.sha256(password.encode()).hexdigest()

    # Fetch the user data from the database
    c.execute('SELECT * FROM users WHERE username=? AND password=?', (username, password_hash))
    user = c.fetchone()

    conn.close()

    if user:
        return True
    else:
        return False

# Initialize session state
if 'login_status' not in st.session_state:
    st.session_state['login_status'] = False


def chatbot():
    st.title("Chatbot Registration & Login")

    # Sidebar for user navigation
    st.sidebar.title("Navigation")
    choice = st.sidebar.radio("Go to", ["Login", "Register"])

    if choice == "Register" and not st.session_state['login_status']:
        st.subheader("Register")
        new_user = st.text_input("Username")
        new_password = st.text_input("Password", type='password')

        if st.button("Register"):
            result = register_user(new_user, new_password)
            st.success(result)

    elif choice == "Login" and not st.session_state['login_status']:
        st.subheader("Login")
        username = st.text_input("Username")
        password = st.text_input("Password", type='password')

        if st.button("Login"):
            login_success = login_user(username, password)
            if login_success:
                st.session_state['login_status'] = True
                st.success("Login successful!")
            else:
                st.error("Login failed. Please check your username and password.")

    if st.session_state['login_status']:
        st.subheader("Welcome to the Chatbot!")
        st.write("You are logged in and the login page is no longer visible.")

if __name__ == "__main__":
    chatbot()

# Load environment variables
google_api = os.getenv('google_api')
# Sidebar
with st.sidebar:
    st.sidebar.title("💬Welcome To The Chatbot World!")
    st.header("Settings")
    theme = st.selectbox("Choose a theme", ["Light", "Dark"])
    st.write("Selected theme:", theme)

# Sidebar - User Profile
st.sidebar.header("User Profile")
# user_avatar = Image.open("legend.jpeg")
# st.sidebar.image(user_avatar, width=100)
st.sidebar.write("**Name:** Dhobi Ved J")
st.sidebar.write("**Status:** Online")
st.sidebar.radio(
    "Select Gender",
    ("male", "female")
)
# Sidebar contents
with st.sidebar:
    st.title('🤗💬 LLM Chat App')
    st.markdown('''
    ## About
    This app is an LLM-powered chatbot built using:
    - [Streamlit](https://streamlit.io/)
    - [LangChain](https://python.langchain.com/)
    - [OpenAI](https://platform.openai.com/docs/models) LLM model
     ''')
    st.write('Made with ❤️ by [Prompt Engineer](https://youtube.com/@engineerprompt)')

load_dotenv()

google_api = os.getenv("AIzaSyB9ds5LORzcW6I2PCPlajCVGU7E9Cy-LrY")
# print(groq_api_key)

st.sidebar.title("Personalization")
prompt = st.sidebar.title("system prompt :")
model = st.sidebar.selectbox(
    'Choose a model',['Llama3-8b-8192','Llama3-70b-8192','Mixtral-8x7b-32768','Gemma-7b-It']
)
#groq client
client = Groq(api_key=google_api)

#streamlit interface
st.title("💬Chat with Groqs llm." "")
st.title("🤖How Can i Help You Today?" "")

#initialize session state for history
if "history" not in st.session_state:
    st.session_state.history = []

user_input = st.text_input("Enter your query:","")
if st.button("submit"):
    chat_completion = client.chat.completions.create(
        messages=[
            {
                "role" : "user",
                "content" : user_input,
            }
        ],
        model = model,
    )
#store the query and response in history
    response = chat_completion.choices[0].message.content
    st.session_state.history.append({"query": user_input,"response":response})

#Display the response
    st.markdown(f'<div class="response-box">{response}</div>',unsafe_allow_html=True)

#Display history
    st.sidebar.title("History")
    for i,entry in enumerate(st.session_state.history):
        if st.sidebar.button(f'query {i+1}:{entry["query"]}'):
            st.markdown(f'<div class="response-box">{entry["response"]}</div>',unsafe_allow_html=True)

import pdfplumber
def extract_text_from_pdf(pdf_file):
    pdf_text = ""
    with pdfplumber.open(pdf_file) as pdf:
        for page in pdf.pages:
            pdf_text += page.extract_text()
    return pdf_text

def answer_question(text, question):
    # Simple keyword-based search
    sentences = text.split('. ')
    sentences = [s.strip() for s in sentences if s]
    best_sentence = max(sentences, key=lambda s: s.lower().count(question.lower()), default="No relevant answer found.")
    return best_sentence

st.title("PDF Chatbot")

# File uploader
pdf_file = st.file_uploader("Upload a PDF", type="pdf")

if pdf_file:
    # Extract text from PDF
    pdf_text = extract_text_from_pdf(pdf_file)
    st.write("PDF text extracted successfully.")

    # Display extracted text (for debugging or confirmation)
    st.text_area("Extracted PDF Text", pdf_text, height=300)

    # Question input
    question = st.text_input("Ask a question about the PDF content:")

    if question:
        # Get answer
        answer = answer_question(pdf_text, question)
        st.write("Answer:", answer)



import streamlit as st
from fpdf import FPDF
from PIL import Image
import os


# Function to create PDF
class PDF(FPDF):
    def header(self):
        self.set_font('Arial', 'B', 12)
        self.cell(0, 10, 'Generated PDF', align='C', ln=1)
        self.ln(10)


def create_pdf(content, images=None, file_name="output.pdf"):
    pdf = PDF()
    pdf.add_page()
    pdf.set_auto_page_break(auto=True, margin=15)

    # Adding text content
    pdf.set_font("Arial", size=12)
    for line in content:
        pdf.multi_cell(0, 10, line)

    # Adding images if any
    if images:
        for img_path in images:
            pdf.add_page()
            pdf.image(img_path, x=10, y=50, w=190)

    pdf.output(file_name)


# Streamlit UI
st.title("Chatbot with PDF Maker")
st.sidebar.header("PDF Settings")

uploaded_files = st.file_uploader("Upload Images (Optional)", type=["png", "jpg", "jpeg"], accept_multiple_files=True)
text_content = st.text_area("Enter your content here:", placeholder="Type text for your PDF...")
layout_option = st.sidebar.selectbox("Select Layout", ["Simple", "With Header"])
file_name = st.sidebar.text_input("File Name", value="output.pdf")

# Ensure 'temp' directory exists
if not os.path.exists("temp"):
    os.makedirs("temp")

# PDF generation
if st.button("Generate PDF"):
    if not text_content.strip() and not uploaded_files:
        st.error("Please provide text or upload at least one image to generate a PDF.")
    else:
        # Process text and images
        text_lines = text_content.split('\n')
        image_paths = []

        if uploaded_files:
            for uploaded_file in uploaded_files:
                try:
                    img = Image.open(uploaded_file)
                    img_path = os.path.join("temp", uploaded_file.name)
                    img.save(img_path)
                    image_paths.append(img_path)
                except Exception as e:
                    st.error(f"Error processing image: {e}")

        # Generate PDF
        try:
            create_pdf(text_lines, images=image_paths, file_name=file_name)
            st.success(f"PDF '{file_name}' has been generated.")
            with open(file_name, "rb") as pdf_file:
                st.download_button(label="Download PDF", data=pdf_file, file_name=file_name, mime="application/pdf")
        except Exception as e:
            st.error(f"Error generating PDF: {e}")

        # Clean up temporary files
        for img_path in image_paths:
            if os.path.exists(img_path):
                os.remove(img_path)
