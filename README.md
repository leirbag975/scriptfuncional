// ============================================
// CONSTANTS & DATA
// ============================================

const ADMIN_UNLOCK_PASSWORD = '490878';
const SCRIPT_ACTIVATION_CODE = '23589';
const STORAGE_KEYS = {
    users: 'scripthub_users',
    scripts: 'scripthub_scripts',
    unlockedScripts: 'scripthub_unlocked',
    adminScripts: 'scripthub_admin_scripts'
};

let currentUser = null;
let adminModeActive = false;
let currentScriptId = null;

// ============================================
// INITIALIZATION
// ============================================

document.addEventListener('DOMContentLoaded', function() {
    initializeApp();
});

function initializeApp() {
    const storedUser = sessionStorage.getItem('currentUser');
    if (storedUser) {
        currentUser = JSON.parse(storedUser);
    }

    if (!localStorage.getItem(STORAGE_KEYS.scripts)) {
        localStorage.setItem(STORAGE_KEYS.scripts, JSON.stringify([]));
    }

    if (!localStorage.getItem(STORAGE_KEYS.users)) {
        localStorage.setItem(STORAGE_KEYS.users, JSON.stringify([]));
    }

    if (!localStorage.getItem(STORAGE_KEYS.unlockedScripts)) {
        localStorage.setItem(STORAGE_KEYS.unlockedScripts, JSON.stringify([]));
    }

    if (!localStorage.getItem(STORAGE_KEYS.adminScripts)) {
        localStorage.setItem(STORAGE_KEYS.adminScripts, JSON.stringify([]));
    }

    loadScripts();
}

// ============================================
// UTILITY FUNCTIONS
// ============================================

function hashPassword(password) {
    let hash = 0;
    for (let i = 0; i < password.length; i++) {
        const char = password.charCodeAt(i);
        hash = ((hash << 5) - hash) + char;
        hash = hash & hash;
    }
    return Math.abs(hash).toString(16);
}

function showNotification(message, type = 'info') {
    const notification = document.getElementById('notification');
    notification.textContent = message;
    notification.className = `notification ${type} show`;

    setTimeout(() => {
        notification.classList.remove('show');
    }, 4000);
}

function generateActivationCode() {
    return Math.random().toString(36).substring(2, 8).toUpperCase();
}

function goToScripts() {
    window.location.href = 'scripts.html';
}

function goToScriptsWithSearch() {
    const search = document.getElementById('heroSearch')?.value || '';
    if (search) {
        localStorage.setItem('searchQuery', search);
    }
    goToScripts();
}

// ============================================
// USER MANAGEMENT
// ============================================

function getUsers() {
    return JSON.parse(localStorage.getItem(STORAGE_KEYS.users) || '[]');
}

function saveUsers(users) {
    localStorage.setItem(STORAGE_KEYS.users, JSON.stringify(users));
}

function getUserByEmail(email) {
    return getUsers().find(user => user.email === email);
}

function clientRegister(event) {
    event.preventDefault();

    const name = document.getElementById('registerName').value;
    const email = document.getElementById('registerEmail').value;
    const password = document.getElementById('registerPassword').value;
    const password2 = document.getElementById('registerPassword2').value;

    if (password !== password2) {
        showNotification('As senhas não coincidem!', 'error');
        return;
    }

    if (getUserByEmail(email)) {
        showNotification('Este email já está cadastrado!', 'error');
        return;
    }

    const users = getUsers();
    const newUser = {
        id: Date.now(),
        name: name,
        email: email,
        password: hashPassword(password),
        createdAt: new Date().toISOString(),
        scripts: []
    };

    users.push(newUser);
    saveUsers(users);

    currentUser = newUser;
    sessionStorage.setItem('currentUser', JSON.stringify(currentUser));

    showNotification('✅ Conta criada com sucesso!', 'success');
    closeRegisterModal();
    updateChatArea();

    document.getElementById('registerName').value = '';
    document.getElementById('registerEmail').value = '';
    document.getElementById('registerPassword').value = '';
    document.getElementById('registerPassword2').value = '';
}

function clientLogin(event) {
    event.preventDefault();

    const email = document.getElementById('loginEmail').value;
    const password = document.getElementById('loginPassword').value;

    const user = getUserByEmail(email);

    if (!user || user.password !== hashPassword(password)) {
        showNotification('Email ou senha incorretos!', 'error');
        return;
    }

    currentUser = user;
    sessionStorage.setItem('currentUser', JSON.stringify(currentUser));

    showNotification('✅ Logado com sucesso!', 'success');
    closeLoginModal();
    updateChatArea();

    document.getElementById('loginEmail').value = '';
    document.getElementById('loginPassword').value = '';
}

function clientLogout() {
    currentUser = null;
    sessionStorage.removeItem('currentUser');
    showNotification('Desconectado!', 'info');
    closeChatArea();
}

// ============================================
// SCRIPTS MANAGEMENT
// ============================================

function getScripts() {
    return JSON.parse(localStorage.getItem(STORAGE_KEYS.scripts) || '[]');
}

function saveScripts(scripts) {
    localStorage.setItem(STORAGE_KEYS.scripts, JSON.stringify(scripts));
}

function getAdminScripts() {
    return JSON.parse(localStorage.getItem(STORAGE_KEYS.adminScripts) || '[]');
}

function saveAdminScripts(scripts) {
    localStorage.setItem(STORAGE_KEYS.adminScripts, JSON.stringify(scripts));
}

function getUnlockedScripts() {
    return JSON.parse(localStorage.getItem(STORAGE_KEYS.unlockedScripts) || '[]');
}

function saveUnlockedScripts(unlocked) {
    localStorage.setItem(STORAGE_KEYS.unlockedScripts, JSON.stringify(unlocked));
}

function loadScripts() {
    const scripts = getScripts();
    const adminScripts = getAdminScripts();
    const allScripts = [...scripts, ...adminScripts];
    const approvedScripts = allScripts.filter(s => s.status === 'approved');
    displayScripts(approvedScripts);
}

function displayScripts(scripts) {
    const grid = document.getElementById('scriptsGrid');
    if (!grid) return;

    const noScripts = document.getElementById('noScripts');

    if (scripts.length === 0) {
        grid.innerHTML = '';
        if (noScripts) noScripts.style.display = 'block';
        return;
    }

    if (noScripts) noScripts.style.display = 'none';

    grid.innerHTML = scripts.map(script => `
        <div class="script-card" onclick="viewScript(${script.id})">
            <img src="${script.image}" alt="${script.name}" class="script-image">
            <div class="script-info">
                <span class="script-category">${script.category}</span>
                <div class="script-name">${script.name}</div>
                <div class="script-description">${script.description}</div>
                <div class="script-author">Por: ${script.authorName}</div>
                <div class="script-footer">
                    <div class="script-price">
                        ${script.price ? 'R$ ' + script.price.toFixed(2) : 'Grátis'}
                    </div>
                    <button class="view-btn">Ver</button>
                </div>
            </div>
        </div>
    `).join('');
}

function filterScripts() {
    const searchTerm = document.getElementById('searchInput')?.value.toLowerCase() || '';
    const category = document.getElementById('categoryFilter')?.value || '';

    let scripts = getScripts();
    let adminScripts = getAdminScripts();
    let allScripts = [...scripts, ...adminScripts];
    allScripts = allScripts.filter(s => s.status === 'approved');

    allScripts = allScripts.filter(script => {
        const matchSearch = script.name.toLowerCase().includes(searchTerm) ||
                          script.description.toLowerCase().includes(searchTerm);
        const matchCategory = !category || script.category === category;
        return matchSearch && matchCategory;
    });

    displayScripts(allScripts);
}

function viewScript(scriptId) {
    let script = getScripts().find(s => s.id === scriptId);
    if (!script) {
        script = getAdminScripts().find(s => s.id === scriptId);
    }

    if (!script) {
        showNotification('Script não encontrado!', 'error');
        return;
    }

    currentScriptId = scriptId;
    const unlocked = getUnlockedScripts().includes(scriptId);

    const page = document.getElementById('scriptDetailPage');
    const container = document.getElementById('scriptDetailContainer');

    let content = `
        <div class="script-detail">
            <img src="${script.image}" alt="${script.name}" class="script-detail-image">
            <div class="script-detail-content">
                <span class="script-detail-category">${script.category}</span>
                <h1 class="script-detail-title">${script.name}</h1>
                <p class="script-detail-author">Por: ${script.authorName}</p>
                <p class="script-detail-description">${script.description}</p>
                <h2 class="script-detail-price">${script.price ? 'R$ ' + script.price.toFixed(2) : 'Grátis'}</h2>

                <div class="script-contact-section">
    `;

    if (unlocked) {
        content += `
                    <h3>✅ Script Desbloqueado</h3>
                    <p>Parabéns! Você desbloqueou este script.</p>
                    <div class="activation-code-box">
                        <span class="activation-code">${script.activationCode}</span>
                    </div>
                    <a href="https://instagram.com/gabriel.futury" target="_blank" class="instagram-redirect-btn">
                        <i class="fab fa-instagram"></i> Ir para Instagram
                    </a>
        `;
    } else {
        content += `
                    <h3>🔒 Script Bloqueado</h3>
                    <p>Digite o código de ativação para desbloquear este script.</p>
                    <button class="btn btn-primary btn-block" onclick="openUnlockModal()">
                        <i class="fas fa-key"></i> Desbloquear Script
                    </button>
        `;
    }

    content += `
                </div>
            </div>
        </div>
    `;

    container.innerHTML = content;
    document.getElementById('scriptsGrid').parentElement.parentElement.style.display = 'none';
    page.style.display = 'block';

    window.scrollTo(0, 0);
}

// ============================================
// CHAT/AREA MODALS
// ============================================

function openChatArea(event) {
    if (event) event.preventDefault();
    document.getElementById('chatModal').classList.add('active');
    updateChatArea();
}

function closeChatArea() {
    document.getElementById('chatModal').classList.remove('active');
}

function switchChatTab(tabName) {
    const tabs = document.querySelectorAll('.chat-tab-content');
    tabs.forEach(tab => tab.classList.remove('active'));

    const buttons = document.querySelectorAll('.chat-tabs .chat-tab-btn');
    buttons.forEach(btn => btn.classList.remove('active'));

    const selectedTab = document.getElementById(`${tabName}-tab`);
    if (selectedTab) selectedTab.classList.add('active');

    if (event && event.target && event.target.classList.contains('chat-tab-btn')) {
        event.target.classList.add('active');
    }
}

function updateChatArea() {
    const myScriptsContent = document.getElementById('myScriptsContent');
    const chatLogoutBtn = document.getElementById('chatLogoutBtn');
    const chatLoginBtn = document.getElementById('chatLoginBtn');

    if (currentUser) {
        chatLogoutBtn.style.display = 'block';
        chatLoginBtn.style.display = 'none';

        const userScripts = getAdminScripts().filter(s => s.userId === currentUser.id && s.status === 'approved');
        
        if (userScripts.length > 0) {
            myScriptsContent.innerHTML = `
                <div style="display: flex; flex-direction: column; gap: 12px;">
                    ${userScripts.map(script => `
                        <div style="background: var(--bg-tertiary); padding: 16px; border-radius: 12px; border: 1px solid var(--border-color);">
                            <img src="${script.image}" alt="${script.name}" style="width: 100%; height: 150px; object-fit: cover; border-radius: 8px; margin-bottom: 12px;">
                            <h4 style="margin-bottom: 6px; color: var(--text-primary);">${script.name}</h4>
                            <p style="font-size: 13px; color: var(--text-secondary); margin-bottom: 12px;">${script.description}</p>
                            <p style="font-size: 12px; color: var(--text-muted); margin-bottom: 12px;">Código: <strong style="color: var(--accent);">${script.activationCode}</strong></p>
                            <button class="btn btn-secondary btn-sm" onclick="deleteAdminScript(${script.id})" style="width: 100%;">
                                <i class="fas fa-trash"></i> Deletar
                            </button>
                        </div>
                    `).join('')}
                </div>
            `;
        } else {
            myScriptsContent.innerHTML = '<p class="info-text">Nenhum script anunciado.</p>';
        }
    } else {
        chatLogoutBtn.style.display = 'none';
        chatLoginBtn.style.display = 'block';
        myScriptsContent.innerHTML = '<p class="info-text">Faça login para ver seus scripts</p>';
    }
}

function deleteAdminScript(scriptId) {
    if (confirm('Tem certeza que deseja deletar este script?')) {
        const scripts = getAdminScripts();
        const index = scripts.findIndex(s => s.id === scriptId);

        if (index > -1) {
            scripts.splice(index, 1);
            saveAdminScripts(scripts);
            showNotification('Script deletado!', 'success');
            updateChatArea();
            loadScripts();
        }
    }
}

// ============================================
// LOGIN/REGISTER MODALS
// ============================================

function openLoginModal() {
    closeChatArea();
    document.getElementById('loginModal').classList.add('active');
}

function closeLoginModal() {
    document.getElementById('loginModal').classList.remove('active');
}

function openRegisterModal() {
    closeLoginModal();
    document.getElementById('registerModal').classList.add('active');
}

function closeRegisterModal() {
    document.getElementById('registerModal').classList.remove('active');
}

// ============================================
// ADMIN PANEL
// ============================================

function openAdminPanel(event) {
    if (event) event.preventDefault();
    document.getElementById('adminModal').classList.add('active');
}

function closeAdminPanel() {
    document.getElementById('adminModal').classList.remove('active');
    adminModeActive = false;
    document.getElementById('adminPassword').value = '';
    document.getElementById('adminPasswordScreen').style.display = 'block';
    document.getElementById('adminPanelScreen').style.display = 'none';
}

function adminPasswordCheck(event) {
    event.preventDefault();

    const password = document.getElementById('adminPassword').value;

    if (password !== ADMIN_UNLOCK_PASSWORD) {
        showNotification('Senha incorreta!', 'error');
        document.getElementById('adminPassword').value = '';
        return;
    }

    adminModeActive = true;
    document.getElementById('adminPasswordScreen').style.display = 'none';
    document.getElementById('adminPanelScreen').style.display = 'block';
    loadAdminScripts();
    showNotification('✅ Acesso concedido!', 'success');
}

function adminLogout() {
    adminModeActive = false;
    document.getElementById('adminPassword').value = '';
    document.getElementById('adminPasswordScreen').style.display = 'block';
    document.getElementById('adminPanelScreen').style.display = 'none';
    document.getElementById('announceForm').reset();
    showNotification('Desconectado do painel!', 'info');
}

function switchAdminTab(tabName) {
    const tabs = document.querySelectorAll('.admin-tab-content');
    tabs.forEach(tab => tab.classList.remove('active'));

    const buttons = document.querySelectorAll('.admin-tabs .admin-tab-btn');
    buttons.forEach(btn => btn.classList.remove('active'));

    const selectedTab = document.getElementById(`${tabName}-tab`);
    if (selectedTab) selectedTab.classList.add('active');

    if (event && event.target && event.target.classList.contains('admin-tab-btn')) {
        event.target.classList.add('active');
    }

    if (tabName === 'manage') {
        loadAdminScripts();
    }
}

function loadAdminScripts() {
    const scripts = getAdminScripts();
    const adminScriptsList = document.getElementById('adminScriptsList');

    if (scripts.length === 0) {
        adminScriptsList.innerHTML = '<p class="info-text">Nenhum script anunciado ainda.</p>';
        return;
    }

    adminScriptsList.innerHTML = scripts.map(script => `
        <div class="admin-item">
            <img src="${script.image}" alt="${script.name}" class="admin-item-image">
            <h4>${script.name}</h4>
            <div class="admin-item-details">
                <p><strong>Categoria:</strong> ${script.category}</p>
                <p><strong>Preço:</strong> ${script.price ? 'R$ ' + script.price.toFixed(2) : 'Grátis'}</p>
                <p><strong>Código:</strong> <span style="color: var(--accent); font-weight: bold;">${script.activationCode}</span></p>
            </div>
            <div class="admin-item-actions">
                <button class="btn btn-secondary btn-sm" onclick="deleteAdminScript(${script.id})">
                    <i class="fas fa-trash"></i> Deletar
                </button>
            </div>
        </div>
    `).join('');
}

function submitScript(event) {
    event.preventDefault();

    const name = document.getElementById('scriptName').value;
    const category = document.getElementById('scriptCategory').value;
    const description = document.getElementById('scriptDescription').value;
    const imageFile = document.getElementById('scriptImage').files[0];
    const price = parseFloat(document.getElementById('scriptPrice').value) || 0;

    if (!imageFile) {
        showNotification('Por favor, selecione uma imagem!', 'error');
        return;
    }

    const reader = new FileReader();
    reader.onload = function(e) {
        const activationCode = generateActivationCode();

        const newScript = {
            id: Date.now(),
            name: name,
            category: category,
            description: description,
            image: e.target.result,
            price: price,
            authorName: 'Admin',
            authorEmail: 'admin@scripthub.com',
            userId: 0,
            status: 'approved',
            activationCode: activationCode,
            createdAt: new Date().toISOString()
        };

        const scripts = getAdminScripts();
        scripts.push(newScript);
        saveAdminScripts(scripts);

        showNotification('🚀 Script anunciado com sucesso! Código: ' + activationCode, 'success');

        document.getElementById('announceForm').reset();
        loadAdminScripts();
        loadScripts();
    };
    reader.readAsDataURL(imageFile);
}

// ============================================
// UNLOCK SCRIPT
// ============================================

function openUnlockModal() {
    document.getElementById('unlockModal').classList.add('active');
}

function closeUnlockModal() {
    document.getElementById('unlockModal').classList.remove('active');
    document.getElementById('unlockCode').value = '';
}

function unlockScript(event) {
    event.preventDefault();

    const code = document.getElementById('unlockCode').value;

    if (code !== SCRIPT_ACTIVATION_CODE) {
        showNotification('Código incorreto!', 'error');
        return;
    }

    const unlocked = getUnlockedScripts();
    if (!unlocked.includes(currentScriptId)) {
        unlocked.push(currentScriptId);
        saveUnlockedScripts(unlocked);
    }

    showNotification('✅ Script desbloqueado com sucesso!', 'success');
    closeUnlockModal();
    viewScript(currentScriptId);
}
