# angelowork
from flask import Flask, render_template, request, jsonify, session
from datetime import datetime
import json
import os

app = Flask(__name__)
app.secret_key = 'your-secret-key-change-this'

# Database semplice (in memoria)
messages = []
users = {}

@app.route('/')
def index():
    return render_template('index.html')

@app.route('/api/register', methods=['POST'])
def register():
    data = request.json
    username = data.get('username')
    
    if username in users:
        return jsonify({'error': 'Username già esistente'}), 400
    
    users[username] = {
        'username': username,
        'joined': datetime.now().isoformat()
    }
    session['username'] = username
    return jsonify({'success': True, 'username': username})

@app.route('/api/login', methods=['POST'])
def login():
    data = request.json
    username = data.get('username')
    
    if username not in users:
        return jsonify({'error': 'Utente non trovato'}), 400
    
    session['username'] = username
    return jsonify({'success': True, 'username': username})

@app.route('/api/logout', methods=['POST'])
def logout():
    session.clear()
    return jsonify({'success': True})

@app.route('/api/send', methods=['POST'])
def send_message():
    if 'username' not in session:
        return jsonify({'error': 'Non autenticato'}), 401
    
    data = request.json
    message = {
        'id': len(messages),
        'sender': session['username'],
        'text': data.get('text'),
        'timestamp': datetime.now().isoformat(),
        'recipient': data.get('recipient', 'all')  # 'all' per chat pubblica
    }
    messages.append(message)
    return jsonify(message)

@app.route('/api/messages', methods=['GET'])
def get_messages():
    if 'username' not in session:
        return jsonify({'error': 'Non autenticato'}), 401
    
    chat_type = request.args.get('chat', 'all')
    
    if chat_type == 'all':
        filtered = [m for m in messages if m['recipient'] == 'all']
    else:
        # Chat privata tra due utenti
        filtered = [m for m in messages 
                   if (m['recipient'] == 'all' or 
                       (m['sender'] == session['username'] and m['recipient'] == chat_type) or
                       (m['sender'] == chat_type and m['recipient'] == session['username']))]
    
    return jsonify(filtered)

@app.route('/api/users', methods=['GET'])
def get_users():
    return jsonify(list(users.keys()))

@app.route('/api/current-user', methods=['GET'])
def get_current_user():
    if 'username' in session:
        return jsonify({'username': session['username']})
    return jsonify({'username': None})

if __name__ == '__main__':
    app.run(debug=True)
