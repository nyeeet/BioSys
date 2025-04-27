    <template>
        <link
        href="https://fonts.googleapis.com/css2?family=Poppins:wght@300;400;500;600;700&display=swap"
        rel="stylesheet"
        />
        <link
        rel="stylesheet"
        href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.4.0/css/all.min.css"
        />
        
        <aside>
        <Sidebar />
        </aside>
        
        <div class="main-content">
        <div class="dashboard-container">
            <AccountHeader />
            <div class="header-actions">
            <h1>SETTINGS</h1>
            </div>
            
            <div class="settings-container">
            <!-- General Settings -->
            <div class="settings-card">
                <div class="card-title">
                <i class="settings-icon fas fa-circle-info"></i>
                General Settings
                </div>
                <div class="form-group">
                <label for="school-name">School Name</label>
                <input type="text" id="school-name" class="form-control" v-model="generalSettings.schoolName">
                </div>
                <div class="form-group">
                <label for="academic-year">Academic Year</label>
                <div class="select-wrapper">
                    <select id="academic-year" class="form-control" v-model="generalSettings.academicYear">
                    <option v-for="year in academicYears" :key="year">{{ year }}</option>
                    </select>
                </div>
                </div>
                <div class="form-group">
                <label for="timezone">Timezone</label>
                <div class="select-wrapper">
                    <select id="timezone" class="form-control" v-model="generalSettings.timezone">
                    <option v-for="tz in timezones" :key="tz">{{ tz }}</option>
                    </select>
                </div>
                </div>
                <button class="save-btn">Save Changes</button>
            </div>
            
            <!-- User Management -->
            <div class="settings-card">
                <div class="card-title">
                <i class="settings-icon fas fa-users"></i>
                User Management
                </div>
                <div class="setting-option">
                <span class="setting-option-label">Allow staff self-registration</span>
                <label class="toggle-switch">
                    <input type="checkbox" v-model="userSettings.allowSelfRegistration">
                    <span class="slider"></span>
                </label>
                </div>
                <div class="setting-option">
                <span class="setting-option-label">Require admin approval for new accounts</span>
                <label class="toggle-switch">
                    <input type="checkbox" v-model="userSettings.requireApproval">
                    <span class="slider"></span>
                </label>
                </div>
                <div class="form-group">
                <label for="default-role">Default Role for New Users</label>
                <div class="select-wrapper">
                    <select id="default-role" class="form-control" v-model="userSettings.defaultRole">
                    <option v-for="role in userRoles" :key="role">{{ role }}</option>
                    </select>
                </div>
                </div>
                <button class="save-btn">Save Changes</button>
            </div>
            
            <!-- Security Settings -->
            <div class="settings-card">
                <div class="card-title">
                <i class="settings-icon fas fa-shield-alt"></i>
                Security Settings
                </div>
                <div class="setting-option">
                <span class="setting-option-label">Enable Two-Factor Authentication</span>
                <label class="toggle-switch">
                    <input type="checkbox" v-model="securitySettings.twoFactorAuth">
                    <span class="slider"></span>
                </label>
                </div>
                <div class="form-group">
                <label for="session-timeout">Session Timeout (minutes)</label>
                <input type="number" id="session-timeout" class="form-control" v-model="securitySettings.sessionTimeout" min="5">
                </div>
                <div class="setting-option">
                <span class="setting-option-label">Log all access attempts</span>
                <label class="toggle-switch">
                    <input type="checkbox" v-model="securitySettings.logAccessAttempts">
                    <span class="slider"></span>
                </label>
                </div>
                <div class="form-group">
                <label for="password-policy">Password Policy</label>
                <div class="select-wrapper">
                    <select id="password-policy" class="form-control" v-model="securitySettings.passwordPolicy">
                    <option v-for="policy in passwordPolicies" :key="policy">{{ policy }}</option>
                    </select>
                </div>
                </div>
                <button class="save-btn">Save Changes</button>
            </div>
            </div>
        </div>
        </div>
    </template>
    
    <script>
    import Sidebar from '../../components/Sidebar.vue'
    import AccountHeader from '../../components/AccountHeader.vue'
    
    export default {
        name: 'SettingsPage',
        components: {
        Sidebar,
        AccountHeader
        },
        data() {
        return {
            // General Settings
            generalSettings: {
            schoolName: '',
            academicYear: '',
            timezone: ''
            },
            academicYears: [],
            timezones: [],
            
            // User Management Settings
            userSettings: {
            allowSelfRegistration: false,
            requireApproval: false,
            defaultRole: ''
            },
            userRoles: [],
            
            // Security Settings
            securitySettings: {
            twoFactorAuth: false,
            sessionTimeout: 30,
            logAccessAttempts: false,
            passwordPolicy: ''
            },
            passwordPolicies: []
        }
        },
        created() {
        // Fetch settings data from API
        this.fetchGeneralSettings();
        this.fetchUserSettings();
        this.fetchSecuritySettings();
        },
        methods: {
        fetchGeneralSettings() {
          
            this.generalSettings = {
            schoolName: '',
            academicYear: '',
            timezone: ''
            };
            
            // Fetch options for dropdowns
            this.academicYears = ['2024-2025', '2025-2026', '2026-2027'];
            this.timezones = [
            'UTC-08:00 Pacific Time',
            'UTC-07:00 Mountain Time',
            'UTC-06:00 Central Time',
            'UTC-05:00 Eastern Time'
            ];
        },
        
        fetchUserSettings() {
            // Replace with actual API call
            this.userSettings = {
            allowSelfRegistration: false,
            requireApproval: true,
            defaultRole: ''
            };
            
            // Fetch roles from API
            this.userRoles = ['Teacher', 'Administrator', 'Office Staff'];
        },
        
        fetchSecuritySettings() {
            
            this.securitySettings = {
            twoFactorAuth: false,
            sessionTimeout: 30,
            logAccessAttempts: true,
            passwordPolicy: ''
            };
            
            // Fetch password policies from API
            this.passwordPolicies = [
            'Standard (8+ chars, 1 number)',
            'Strong (10+ chars, mixed case, numbers)',
            'Very Strong (12+ chars, mixed case, numbers, symbols)'
            ];
        },
        
        saveGeneralSettings() {
            // Implementation for saving general settings to API
            console.log('Saving general settings:', this.generalSettings);
         
            this.showSaveSuccess('General');
        },
        
        saveUserSettings() {
            // Implementation for saving user settings 
            console.log('Saving user settings:', this.userSettings);
            // API call would go here
            this.showSaveSuccess('User');
        },
        
        saveSecuritySettings() {
            // Implementation for saving security settings 
            console.log('Saving security settings:', this.securitySettings);
         
            this.showSaveSuccess('Security');
        },
        
        showSaveSuccess(settingType) {
            // Simple alert but could be replaced with a toast notification
            alert(`${settingType} settings saved successfully!`);
        }
        }
    }
    </script>
    
    <style scoped>
    .main-content {
        display: flex;
        flex-direction: column;
        align-items: end;
        width: 100%;
    }
    
    .dashboard-container {
        width: 95%;
    }
    
    .header-actions {
        margin-bottom: 20px;
    }
    
    .header-actions h1 {
        font-family: 'Poppins', sans-serif;
        font-size: 40px;
        text-align: center;
    }
    
    .settings-container {
        display: grid;
        margin-left: 12px;
        margin-right: 12px;
        gap: 10px;
        margin-bottom: 30px;
    }
    
    .settings-card {
        background-color: #1e3a5f;
        border-radius: 8px;
        padding: 24px;
        box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
        color: #ffffff;
        font-family: 'Poppins', sans-serif;
        margin-bottom: 20px;
    }
    
    .card-title {
        font-size: 20px;
        font-weight: bold;
        margin-bottom: 24px;
        display: flex;
        align-items: center;
        font-family: 'Poppins', sans-serif;
    }
    
    .settings-icon {
        margin-right: 10px;
        font-size: 18px;
    }
    
    .form-group {
        margin-bottom: 20px;
    }
    
    .form-group label {
        display: block;
        margin-bottom: 8px;
        font-weight: 500;
        font-family: 'Poppins', sans-serif;
    }
    
    .form-control {
        width: 100%;
        padding: 12px;
        border: none;
        border-radius: 4px;
        background-color: #2a4d7d;
        color: #ffffff;
        box-sizing: border-box;
        font-family: 'Poppins', sans-serif;
    }
    
    .select-wrapper {
        position: relative;
    }
    
    .select-wrapper::after {
        content: "";
        position: absolute;
        right: 12px;
        top: 50%;
        transform: translateY(-50%);
        width: 0;
        height: 0;
        border-left: 6px solid transparent;
        border-right: 6px solid transparent;
        border-top: 6px solid #ffffff;
        pointer-events: none;
    }
    
    select.form-control {
        appearance: none;
        padding-right: 30px;
    }
    
    .toggle-switch {
        position: relative;
        display: inline-block;
        width: 60px;
        height: 30px;
    }
    
    .toggle-switch input {
        opacity: 0;
        width: 0;
        height: 0;
    }
    
    .slider {
        position: absolute;
        cursor: pointer;
        top: 0;
        left: 0;
        right: 0;
        bottom: 0;
        background-color: #cccccc;
        transition: .3s;
        border-radius: 30px;
    }
    
    .slider:before {
        position: absolute;
        content: "";
        height: 24px;
        width: 24px;
        left: 3px;
        bottom: 3px;
        background-color: white;
        transition: .3s;
        border-radius: 50%;
    }
    
    input:checked + .slider {
        background-color: #4CAF50;
    }
    
    input:checked + .slider:before {
        transform: translateX(30px);
    }
    
    .setting-option {
        display: flex;
        justify-content: space-between;
        align-items: center;
        margin-bottom: 20px;
    }
    
    .setting-option-label {
        font-weight: 500;
        font-family: 'Poppins', sans-serif;
    }
    
    .save-btn {
        background-color: #4CAF50;
        color: white;
        border: none;
        padding: 12px 24px;
        border-radius: 4px;
        cursor: pointer;
        font-weight: 500;
        font-family: 'Poppins', sans-serif;
        transition: background-color 0.3s;
    }
    
    .save-btn:hover {
        background-color: #3d8b40;
    }
    

    </style>