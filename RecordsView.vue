  <script setup>
  import { ref, computed, onMounted, watch } from 'vue'
  import DeleteRecordModal from '../../components/DeleteRecordModal.vue'
  import { useRoute, useRouter } from 'vue-router'

  const route = useRoute()
  const router = useRouter()

  const records = ref([])
  const isLoading = ref(true)
  const error = ref(null)


  const itemsPerPage = 10
  const currentPage = ref(1)

  const props = defineProps({
    sectionId: {
      type: String,
      default: ''
    },
    sectionData: {
      type: Object,
      default: () => ({})
    }
  })

  const sectionId = computed(() => route.params.id)
  const sectionData = ref({})

  onMounted(() => {
    if (route.query.sectionData) {
      try {
        sectionData.value = JSON.parse(decodeURIComponent(route.query.sectionData))
      } catch (err) {
        console.error('Error parsing section data:', err)
      }
    }
    
    fetchRecords()
  })

  const fetchRecords = async () => {
    isLoading.value = true
    error.value = null
    
    try {
      const endpoint = sectionId.value 
        ? `/api/attendance-records?sectionId=${sectionId.value}` 
        : '/api/attendance-records'
        
      records.value = generateSampleRecords(sectionData.value)
      calculatePreviousPeriodCounts()
    } catch (err) {
      console.error('Error fetching records:', err)
      error.value = err.message
      records.value = []
    } finally {
      isLoading.value = false
    }
  }

  const generateSampleRecords = (section) => {
    const statuses = [
      'checkin_am', 'checkin_pm', 'checkout_am', 'checkout_pm',
      'absent_am', 'absent_pm', 'late_am', 'late_pm'
    ]
    
    const students = []
    
    const records = []
    
    for (let i = 0; i < 20; i++) {
      const date = new Date()
      date.setDate(date.getDate() - Math.floor(Math.random() * 14))
      
      const hours = Math.floor(Math.random() * 8) + 8
      const minutes = Math.floor(Math.random() * 60)
      
      records.push({
        id: `record-${i + 1}`,
        name: students[Math.floor(Math.random() * students.length)],
        level: section?.projectName || 'Grade ' + (Math.floor(Math.random() * 12) + 1),
        section: section?.section || 'Section ' + String.fromCharCode(65 + Math.floor(Math.random() * 3)),
        date: date.toLocaleDateString(),
        time: `${hours.toString().padStart(2, '0')}:${minutes.toString().padStart(2, '0')}`,
        status: statuses[Math.floor(Math.random() * statuses.length)]
      })
    }
    
    return records
  }

  const previousPeriodCounts = ref({
    checkin_am: 0,
    checkin_pm: 0, 
    checkout_am: 0,
    checkout_pm: 0,
    absent_am: 0,
    absent_pm: 0,
    late_am: 0,
    late_pm: 0
  })

  const calculatePreviousPeriodCounts = async () => {
    try {
      const currentCounts = statusCounts.value
      
      for (const key in currentCounts) {
        previousPeriodCounts.value[key] = Math.max(1, Math.floor(currentCounts[key] * 0.9))
      }
    } catch (err) {
      console.error('Error fetching previous period data:', err)
    }
  }

  const statusCounts = computed(() => {
    return {
      checkin_am: records.value.filter(r => r.status === 'checkin_am').length,
      checkin_pm: records.value.filter(r => r.status === 'checkin_pm').length,
      checkout_am: records.value.filter(r => r.status === 'checkout_am').length,
      checkout_pm: records.value.filter(r => r.status === 'checkout_pm').length,
      absent_am: records.value.filter(r => r.status === 'absent_am').length,
      absent_pm: records.value.filter(r => r.status === 'absent_pm').length,
      late_am: records.value.filter(r => r.status === 'late_am').length,
      late_pm: records.value.filter(r => r.status === 'late_pm').length
    }
  })

  const statusTrends = computed(() => {
    const calculateTrend = (current, previous) => {
      if (previous === 0) return current > 0 ? 100 : 0; 
      return Math.round(((current - previous) / previous) * 100);
    };

    return {
      checkin_am: calculateTrend(statusCounts.value.checkin_am, previousPeriodCounts.value.checkin_am),
      checkin_pm: calculateTrend(statusCounts.value.checkin_pm, previousPeriodCounts.value.checkin_pm),
      checkout_am: calculateTrend(statusCounts.value.checkout_am, previousPeriodCounts.value.checkout_am),
      checkout_pm: calculateTrend(statusCounts.value.checkout_pm, previousPeriodCounts.value.checkout_pm),
      absent_am: calculateTrend(statusCounts.value.absent_am, previousPeriodCounts.value.absent_am),
      absent_pm: calculateTrend(statusCounts.value.absent_pm, previousPeriodCounts.value.absent_pm),
      late_am: calculateTrend(statusCounts.value.late_am, previousPeriodCounts.value.late_am),
      late_pm: calculateTrend(statusCounts.value.late_pm, previousPeriodCounts.value.late_pm)
    }
  })

  const activeFilter = ref(null)
  const sortBy = ref('date-newest')
  const studentSearch = ref('')
  const sectionSearch = ref('')

  const showDeleteModal = ref(false)
  const recordToDelete = ref({})

  const sortedFilteredRecords = computed(() => {
    let result = activeFilter.value ? 
      records.value.filter(record => record.status === activeFilter.value) : 
      [...records.value]
    
    if (studentSearch.value.trim()) {
      const searchTerm = studentSearch.value.toLowerCase().trim()
      result = result.filter(record => 
        record.name.toLowerCase().includes(searchTerm)
      )
    }
    
    if (sectionSearch.value.trim()) {
      const searchTerm = sectionSearch.value.toLowerCase().trim()
      result = result.filter(record => 
        record.level.toLowerCase().includes(searchTerm)
      )
    }
    
    switch (sortBy.value) {
      case 'date-newest':
        result.sort((a, b) => new Date(b.date) - new Date(a.date))
        break
      case 'date-oldest':
        result.sort((a, b) => new Date(a.date) - new Date(b.date))
        break
      case 'name-az':
        result.sort((a, b) => a.name.localeCompare(b.name))
        break
      case 'name-za':
        result.sort((a, b) => b.name.localeCompare(a.name))
        break
      case 'level':
        result.sort((a, b) => a.level.localeCompare(b.level))
        break
      case 'status':
        result.sort((a, b) => a.status.localeCompare(b.status))
        break
    }
    
    return result
  })

  const totalPages = computed(() => {
    return Math.ceil(sortedFilteredRecords.value.length / itemsPerPage)
  })

  const paginatedRecords = computed(() => {
    const startIndex = (currentPage.value - 1) * itemsPerPage
    const endIndex = startIndex + itemsPerPage
    return sortedFilteredRecords.value.slice(startIndex, endIndex)
  })

  const showingStart = computed(() => {
    if (sortedFilteredRecords.value.length === 0) return 0
    return (currentPage.value - 1) * itemsPerPage + 1
  })

  const showingEnd = computed(() => {
    return Math.min(currentPage.value * itemsPerPage, sortedFilteredRecords.value.length)
  })

  const goToPage = (page) => {
    if (page >= 1 && page <= totalPages.value) {
      currentPage.value = page
    }
  }

  const goToFirstPage = () => {
    currentPage.value = 1
  }

  const goToLastPage = () => {
    currentPage.value = totalPages.value
  }

  const goToPrevPage = () => {
    if (currentPage.value > 1) {
      currentPage.value--
    }
  }

  const goToNextPage = () => {
    if (currentPage.value < totalPages.value) {
      currentPage.value++
    }
  }

  // Generate page numbers for pagination
  const pageNumbers = computed(() => {
    const pages = []
    let startPage = Math.max(1, currentPage.value - 1)
    let endPage = Math.min(totalPages.value, startPage + 2)
    
    // Adjust if at the end of the range
    if (endPage === totalPages.value) {
      startPage = Math.max(1, endPage - 2)
    }
    
    for (let i = startPage; i <= endPage; i++) {
      pages.push(i)
    }
    
    return pages
  })

  // Watch for filter changes to reset pagination
  watch([activeFilter, studentSearch, sectionSearch, sortBy], () => {
    currentPage.value = 1
  })

  const setFilter = (filter) => {
    if (activeFilter.value === filter) {
      activeFilter.value = null
    } else {
      activeFilter.value = filter
    }
  }

  const clearAllFilters = () => {
    activeFilter.value = null
    studentSearch.value = ''
    sectionSearch.value = ''
  }

  const openDeleteModal = (record) => {
    recordToDelete.value = record
    showDeleteModal.value = true
  }

  const confirmDelete = async (recordId) => {
    try {
      const recordToRemove = records.value.find(r => r.id === recordId)
      const statusToUpdate = recordToRemove?.status
    
      const index = records.value.findIndex(r => r.id === recordId)
      if (index !== -1) {
        records.value.splice(index, 1)
        
        if (statusToUpdate) {
          previousPeriodCounts.value[statusToUpdate] = Math.max(1, previousPeriodCounts.value[statusToUpdate] - 1)
        }
        
        showNotification('Record deleted successfully')
      }
    } catch (err) {
      console.error('Error deleting record:', err)
      showNotification('Failed to delete record', 'error')
    }
  }

  const showDeleteRecordModal = ref(false)
  const openDeleteRecordModal = (record) => {
    recordToDelete.value = record
    showDeleteRecordModal.value = true
  }

  const notification = ref({ show: false, message: '', type: 'success' })

  const showNotification = (message, type = 'success') => {
    notification.value = { show: true, message, type }
    
    setTimeout(() => {
      notification.value.show = false
    }, 3000)
  }

  const refreshData = async () => {
    await fetchRecords()
    showNotification('Data refreshed successfully')
  }

  const goBackToSections = () => {
    router.push('/sections')
  }
  </script>

  <template>
    <link
      href="https://fonts.googleapis.com/css2?family=Poppins:wght@300;400;500;600;700&display=swap"
      rel="stylesheet"
    />
    <link
      rel="stylesheet"
      href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.4.0/css/all.min.css"
    />

    <div class="main-content">
      <div class="dashboard-container">
        <div class="header-actions">
          <h1>ATTENDANCE RECORDS</h1>
        
          <div class="section-info">
            <span class="section-details">
              <strong>Grade Level:</strong> {{ sectionData.projectName || 'All Grades' }} |
              <strong>Section:</strong> {{ sectionData.section || 'All Sections' }}
            </span>
          </div>

        </div>
        
        <div v-if="error" class="error-message">
          <i class="fas fa-exclamation-triangle"></i>
          Failed to load records: {{ error }}
          <button @click="fetchRecords" class="retry-btn">
            <i class="fas fa-redo"></i> Retry
          </button>
        </div>
    
        <div v-else>
          <div class="status-summary">
            <div 
              class="status-card" 
              :class="{ 'active-filter': activeFilter === 'checkin_am' }"
              @click="setFilter('checkin_am')"
            >
              <div class="status-icon checkin">
                <span class="status-label">AM Check-ins</span>
                <i class="fas fa-check"></i>
              </div>
              <div class="status-number">{{ statusCounts.checkin_am }}</div>
              <div :class="['status-trend', statusTrends.checkin_am >= 0 ? 'trend-up' : 'trend-down']">
                <i :class="['fas', statusTrends.checkin_am >= 0 ? 'fa-arrow-up' : 'fa-arrow-down']"></i> 
                {{ Math.abs(statusTrends.checkin_am) }}% from previous period
              </div>
            </div>
          
            <div 
              class="status-card" 
              :class="{ 'active-filter': activeFilter === 'checkout_am' }"
              @click="setFilter('checkout_am')"
            >
              <div class="status-icon checkout">
                <span class="status-label">AM Check-outs</span>
                <i class="fas fa-sign-out-alt"></i>
              </div>
              <div class="status-number">{{ statusCounts.checkout_am }}</div>
              <div :class="['status-trend', statusTrends.checkout_am >= 0 ? 'trend-up' : 'trend-down']">
                <i :class="['fas', statusTrends.checkout_am >= 0 ? 'fa-arrow-up' : 'fa-arrow-down']"></i> 
                {{ Math.abs(statusTrends.checkout_am) }}% from previous period
              </div>
            </div>
            
            <div 
              class="status-card" 
              :class="{ 'active-filter': activeFilter === 'late_am' }"
              @click="setFilter('late_am')"
            >
              <div class="status-icon late">
                <span class="status-label">Late AM Check-ins</span>
                <i class="fas fa-clock"></i>
              </div>
              <div class="status-number">{{ statusCounts.late_am }}</div>
              <div :class="['status-trend', statusTrends.late_am >= 0 ? 'trend-up' : 'trend-down']">
                <i :class="['fas', statusTrends.late_am >= 0 ? 'fa-arrow-up' : 'fa-arrow-down']"></i> 
                {{ Math.abs(statusTrends.late_am) }}% from previous period
              </div>
            </div>
      
            <div 
              class="status-card" 
              :class="{ 'active-filter': activeFilter === 'absent_am' }"
              @click="setFilter('absent_am')"
            >
              <div class="status-icon absent">
                <span class="status-label">AM Absent</span>
                <i class="fas fa-times"></i>
              </div>
              <div class="status-number">{{ statusCounts.absent_am }}</div>
              <div :class="['status-trend', statusTrends.absent_am >= 0 ? 'trend-up' : 'trend-down']">
                <i :class="['fas', statusTrends.absent_am >= 0 ? 'fa-arrow-up' : 'fa-arrow-down']"></i> 
                {{ Math.abs(statusTrends.absent_am) }}% from previous period
              </div>
            </div>
          </div>
          
          <div class="status-summary">
            <div 
              class="status-card" 
              :class="{ 'active-filter': activeFilter === 'checkin_pm' }"
              @click="setFilter('checkin_pm')"
            >
              <div class="status-icon checkin">
                <span class="status-label">PM Check-ins</span>
                <i class="fas fa-check"></i>
              </div>
              <div class="status-number">{{ statusCounts.checkin_pm }}</div>
              <div :class="['status-trend', statusTrends.checkin_pm >= 0 ? 'trend-up' : 'trend-down']">
                <i :class="['fas', statusTrends.checkin_pm >= 0 ? 'fa-arrow-up' : 'fa-arrow-down']"></i> 
                {{ Math.abs(statusTrends.checkin_pm) }}% from previous period
              </div>
            </div>
            
            <div 
              class="status-card" 
              :class="{ 'active-filter': activeFilter === 'checkout_pm' }"
              @click="setFilter('checkout_pm')"
            >
              <div class="status-icon checkout">
                <span class="status-label">PM Check-outs</span>
                <i class="fas fa-sign-out-alt"></i>
              </div>
              <div class="status-number">{{ statusCounts.checkout_pm }}</div>
              <div :class="['status-trend', statusTrends.checkout_pm >= 0 ? 'trend-up' : 'trend-down']">
                <i :class="['fas', statusTrends.checkout_pm >= 0 ? 'fa-arrow-up' : 'fa-arrow-down']"></i> 
                {{ Math.abs(statusTrends.checkout_pm) }}% from previous period
              </div>
            </div>        
          
            <div 
              class="status-card" 
              :class="{ 'active-filter': activeFilter === 'late_pm' }"
              @click="setFilter('late_pm')"
            >
              <div class="status-icon late">
                <span class="status-label">Late PM Check-ins</span>
                <i class="fas fa-clock"></i>
              </div>
              <div class="status-number">{{ statusCounts.late_pm }}</div>
              <div :class="['status-trend', statusTrends.late_pm >= 0 ? 'trend-up' : 'trend-down']">
                <i :class="['fas', statusTrends.late_pm >= 0 ? 'fa-arrow-up' : 'fa-arrow-down']"></i> 
                {{ Math.abs(statusTrends.late_pm) }}% from previous period
              </div>
            </div>
            
            <div 
              class="status-card" 
              :class="{ 'active-filter': activeFilter === 'absent_pm' }"
              @click="setFilter('absent_pm')"
            >
              <div class="status-icon absent">
                <span class="status-label">PM Absent</span>
                <i class="fas fa-times"></i>
              </div>
              <div class="status-number">{{ statusCounts.absent_pm }}</div>
              <div :class="['status-trend', statusTrends.absent_pm >= 0 ? 'trend-up' : 'trend-down']">
                <i :class="['fas', statusTrends.absent_pm >= 0 ? 'fa-arrow-up' : 'fa-arrow-down']"></i> 
                {{ Math.abs(statusTrends.absent_pm) }}% from previous period
              </div>
            </div>
          </div>
                <div class="sort-search">
          <div class="search-controls">
            <div class="search-input">
              <i class="fas fa-search"></i>
              <input 
                type="text" 
                v-model="studentSearch" 
                placeholder="Search student name..."
              />
              <button v-if="studentSearch" @click="studentSearch = ''" class="clear-search">
                <i class="fas fa-times"></i>
              </button>
            </div>
          </div>
          <div class="sort-controls">
            <span><i class="fas fa-sort"></i> Sort by:</span>
            <select class="sort-select" v-model="sortBy">
              <option value="date-newest">Date (Newest First)</option>
              <option value="date-oldest">Date (Oldest First)</option>
              <option value="name-az">Student Name (A-Z)</option>
              <option value="name-za">Student Name (Z-A)</option>
              <option value="level">Level</option>
              <option value="status">Status</option>
            </select>
          </div>
        </div>
          <div v-if="activeFilter || studentSearch || sectionSearch" class="filter-indicator">
            <span>
              <template v-if="activeFilter">
                Filtered by status: 
                <strong>{{ {
                  'checkin_am': 'AM Check-in',
                  'checkin_pm': 'PM Check-in',
                  'checkout_am': 'AM Check-out',
                  'checkout_pm': 'PM Check-out',
                  'absent_am': 'AM Absent',
                  'absent_pm': 'PM Absent',
                  'late_am': 'Late AM Check-in',
                  'late_pm': 'Late PM Check-in'
                }[activeFilter] }}</strong>
              </template>
              <template v-if="studentSearch">
                <template v-if="activeFilter"> | </template>
                Student search: <strong>"{{ studentSearch }}"</strong>
              </template>
              <template v-if="sectionSearch">
                <template v-if="activeFilter || studentSearch"> | </template>
                Section search: <strong>"{{ sectionSearch }}"</strong>
              </template>
            </span>
            <button @click="clearAllFilters()" class="clear-filter-btn">
              <i class="fas fa-times"></i> Clear all filters
            </button>
          </div>
          
          <table class="records-table">
            <thead>
              <tr>
                <th>STUDENT</th>
                <th>LEVEL</th>
                <th>DATE</th>
                <th>STATUS</th>
                <th>TIME</th>
                <th>ACTIONS</th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="record in paginatedRecords" :key="record.id">
                <td>{{ record.name }}</td>
                <td>{{ record.level }}</td>
                <td>{{ record.date }}</td>
                <td>
                  <span :class="['status-badge', record.status.split('_')[0]]">
                    <i :class="[
                      'fas', 
                      record.status.includes('checkin') ? 'fa-check' : 
                      record.status.includes('checkout') ? 'fa-sign-out-alt' : 
                      record.status.includes('absent') ? 'fa-times' : 'fa-clock'
                    ]"></i>
                    {{ {
                      'checkin_am': 'AM Check-in',
                      'checkin_pm': 'PM Check-in',
                      'checkout_am': 'AM Check-out',
                      'checkout_pm': 'PM Check-out',
                      'absent_am': 'AM Absent',
                      'absent_pm': 'PM Absent',
                      'late_am': 'Late AM Check-in',
                      'late_pm': 'Late PM Check-in'
                    }[record.status] }}
                  </span>
                </td>
                <td>{{ record.time }}</td>
                <td>
                  <span class="action-btn delete-btn" @click="openDeleteRecordModal(record)">
                    <i class="fas fa-trash"></i>
                  </span>
                </td>
              </tr>
              <tr v-if="sortedFilteredRecords.length === 0 && !isLoading">
                <td colspan="6" class="no-records">No records found</td>
              </tr>
            </tbody>
          </table>
          
          <div class="pagination-container">
            <div>Showing {{ showingStart }} to {{ showingEnd }} of {{ sortedFilteredRecords.length }} records</div>
            <div class="pagination">
              <button @click="goToFirstPage" :disabled="currentPage === 1">
                <i class="fas fa-angle-double-left"></i>
              </button>
              <button @click="goToPrevPage" :disabled="currentPage === 1">
                <i class="fas fa-angle-left"></i>
              </button>
              <button 
                v-for="page in pageNumbers" 
                :key="page" 
                :class="{ 'active': currentPage === page }"
                @click="goToPage(page)"
              >
                {{ page }}
              </button>
              <button @click="goToNextPage" :disabled="currentPage === totalPages">
                <i class="fas fa-angle-right"></i>
              </button>
              <button @click="goToLastPage" :disabled="currentPage === totalPages">
                <i class="fas fa-angle-double-right"></i>
              </button>
            </div>
          </div>
        </div>

        <div v-if="notification.show" :class="['notification', notification.type]">
          <i :class="['fas', notification.type === 'success' ? 'fa-check-circle' : 'fa-exclamation-circle']"></i>
          {{ notification.message }}
        </div>
      </div>
    </div>
    
    <DeleteRecordModal
      :is-open="showDeleteRecordModal"
      :record="recordToDelete"
      @close="showDeleteRecordModal = false"
      @confirm="confirmDelete"
    />
  </template>
    
    <style scoped>
    .header-actions {
      display: flex;
      justify-content: space-between;
      align-items: center;
      margin-bottom: 1rem;
    }
    
    .refresh-btn {
      background-color: #f8f9fa;
      border: 1px solid #ddd;
      border-radius: 4px;
      padding: 0.5rem 1rem;
      cursor: pointer;
      display: flex;
      align-items: center;
      gap: 0.5rem;
      transition: all 0.2s;
    }
    
    .refresh-btn:hover {
      background-color: #e9ecef;
    }
    
    .error-message {
      background-color: #fff3f3;
      color: #e74c3c;
      padding: 1rem;
      border-radius: 4px;
      margin-bottom: 1rem;
      display: flex;
      align-items: center;
      gap: 0.5rem;
    }
    
    .retry-btn {
      margin-left: auto;
      background-color: #e74c3c;
      color: white;
      border: none;
      border-radius: 4px;
      padding: 0.3rem 0.8rem;
      cursor: pointer;
      display: flex;
      align-items: center;
      gap: 0.3rem;
    }
    
    .loading-indicator {
      text-align: center;
      padding: 2rem;
      color: #6c757d;
      font-size: 1rem;
    }
    
    .loading-indicator i {
      margin-right: 0.5rem;
    }
    
    .sort-search {
      display: flex;
      justify-content: space-between;
      align-items: center;
      margin-bottom: 1.5rem;
      margin: auto;
      flex-wrap: wrap;
      gap: 1rem;
    }
    
    .search-controls {
      display: flex;
      gap: 1rem;
      flex: 1;
      flex-wrap: wrap;
      margin-right: 0;
    }
    
    .search-input {
      position: relative;
      flex: 1;
      min-width: 60%;
    }
    
    .search-input input {
      width: 85%;
      padding: 0.5rem 2rem 0.5rem 2rem;
      border-radius: 4px;
      border: 1px solid #ddd;
      font-size: 0.9rem;
    }
    
    .search-input i {
      position: absolute;
      left: 0.6rem;
      top: 50%;
      transform: translateY(-50%);
      color: #666;
    }
    
    .clear-search {
      position: absolute;
      right: 3.0rem;
      top: 50%;
      transform: translateY(-50%);
      background: transparent;
      border: none;
      color: #666;
      cursor: pointer;
    }
    
    .clear-search:hover {
      color: #333;
    }
    
    .notification {
      position: fixed;
      padding: 12px 10px;
      border-radius: 4px;
      display: flex;
      gap: 0.6rem;
      box-shadow: 0 4px 12px rgba(0, 0, 0, 0.15);
      z-index: 1000;
      animation: slideIn 0.3s ease-out;
    }
    
    .notification.success {
      background-color: #e7f7f0;
      color: #0a7248;
      border-left: 4px solid #0a7248;
    }
    
    .notification.error {
      background-color: #fbeaea;
      color: rgb(213, 213, 92);
      border-left: 4px solid #d63031;
    }

  .menu-item {
    display: flex;
    align-items: center;
    border-radius: 5px;
    cursor: pointer;
  }

  .menu-item.active {
    background-color: #2c5f8e;
    font-weight: bold;
  }

  .menu-item:not(.active) {
    background-color: #1a3148;
  }

  .menu-item:hover:not(.active) {
    background-color: #224262;
  }

  .menu-icon {
    width: 24px;
    height: 20px;
    border: 2px solid white;
    margin-right: 15px;
  }

  .main-content {
    width: 98%;
    height: auto;
    margin-left: 15px;
    

  }

  .status-filters {
    display: flex;
    gap: 10px;
  }

  .sort-search {
    display: flex;
    gap: 15px;
    margin-bottom: 20px;
  }

  .sort-controls {
    padding: 10px;
    
    border-radius: 5px;
    display: flex;
    align-items: center;
    flex: 1;
  }

  .sort-select {
    background-color: white;
    border: 1px solid #234666;
    color: #1a3148;
    box-shadow: 0 2px 10px rgba(0, 0, 0, 0.2);
    padding: 5px 10px;
    margin: 0 10px;
    border-radius: 30px;
    min-width: 180px;
    cursor: pointer;
    padding: 10px 20px;
    font-size: 16px;
    border: 0;
  }

  .sort-select:hover {
    background-color: white;
    box-shadow: 0 4px 12px rgba(0, 0, 0, 0.3);
    
  }

  .sort-select:focus {
    outline: none;
    box-shadow: 0 0 0 2px rgba(33, 150, 243, 0.5);
  }

  .sort-select option {
    background-color:  rgba(242, 242, 242, 0.9);
    color: #1a3148;
    padding: 10px;
    cursor: pointer;
  }

  .search-box {
    background-color: #15273a;
    padding: 10px;
    border-radius: 5px;
    display: flex;
    align-items: center;
  }

  .search-box input {
    background-color: #1a3148;
    border: 1px solid white;
    color: #aaa;
    padding: 5px 10px;
    margin-right: 10px;
    border-radius: 3px;
    width: 200px;
  }

  .search-btn {
    background-color: #4caf50;
    color: white;
    border: none;
    padding: 7px 12px;
    border-radius: 3px;
    cursor: pointer;
  }

  .status-summary {
    display: flex;
    justify-content: space-between;
    gap: 20px;
    margin-bottom: 20px;
  }

  .status-card {
    background-color: #1a3550;
    padding: 12px 16px 15px 25px;
    border-radius: 15px;
    display: flex;
    flex-direction: column;
    flex: 1;
    box-shadow: 0 2px 10px rgba(0, 0, 0, 0.2);
    transition: transform 0.3s ease;
    text-align: left;
  }

  .status-card:hover {
    transform: translateY(-5px);
    cursor: pointer;
  }

  .status-card.invisible {
    visibility: hidden;
  }

  .status-icon {
    display: flex;
    justify-content: space-between;
    width: 100%;
    align-items: center;
    margin-bottom: 15px;
  }

  .status-icon i {
    font-size: 24px;
    border-radius: 50%;
    width: 40px;
    height: 40px;
    display: flex;
    align-items: center;
    justify-content: center;
  }

  .status-icon.checkin i {
    background-color: #4caf50;
  }

  .status-icon.checkout i {
    background-color: #2196f3;
  }

  .status-icon.absent i {
    background-color: #f44336;
  }

  .status-icon.late i {
    background-color: #ff9800;
  }

  .status-number {
    font-size: 24px;
    font-weight: bold;
  }

  .status-label {
    color: #aaa;
    font-size: 18px;
    margin: 0;
    font-weight: normal;
  }

  .status-trend {
    font-size: 14px;
    display: flex;
    align-items: center;
    gap: 3px;
  }

  .trend-up {
    color: #4caf50;
  }

  .trend-down {
    color: #f44336;
  }

  .filter-indicator {
    background-color: rgba(76, 175, 80, 0.1);
    border: 1px solid rgba(76, 175, 80, 0.3);
    padding: 10px 15px;
    border-radius: 5px;
    margin-bottom: 15px;
    display: flex;
    justify-content: space-between;
    align-items: center;
  }

  .clear-filter-btn {
    background-color: transparent;
    border: 1px solid #4caf50;
    color: #4caf50;
    padding: 5px 10px;
    border-radius: 3px;
    cursor: pointer;
    display: flex;
    align-items: center;
    gap: 5px;
  }

  .clear-filter-btn:hover {
    background-color: rgba(76, 175, 80, 0.1);
  }

  .records-table {
    width: 100%;
    border-collapse: collapse;
    margin-bottom: 20px;
    border-radius: 8px;
    overflow: hidden;
  }

  .records-table th {
    background-color: #234666;
    padding: 12px 15px;
    text-align: left;
    font-weight: 500;
    color: white;
    text-transform: uppercase;
    font-size: 14px;
    border-bottom: 1px solid #0e2941;
  }

  .records-table th.sortable {
    cursor: pointer;
    user-select: none;
  }

  .records-table th.sortable:hover {
    background-color: #1a3550;
  }

  .records-table th i {
    margin-left: 5px;
    font-size: 12px;
  }

  .records-table td {
    background-color: rgba(242, 242, 242, 0.9);
    color: #0e2941;
    padding: 12px 15px;
    border-bottom: 1px solid #0e2941;
  }

  .records-table tr:last-child td {
    border-bottom: none;
  }

  .records-table tr:hover td {
    background-color: rgba(242, 242, 242, 1);
  }
  .notification {
    position: fixed;
    top: 10%;
    left: 50%;
    transform: translateX(-50%);
    padding: 10px 20px;
    border-radius: 5px;
    display: flex;
    align-items: center;
    gap: 10px;
    color: white;
    font-weight: 500;
    z-index: 1000;
    box-shadow: 0 3px 6px rgba(0, 0, 0, 0.16);
    animation: fadeIn 0.3s ease;
  }

  .notification.success {
    background-color: #4CAF50;
  }

  .notification.error {
    background-color: #F44336;
  }

  @keyframes fadeIn {
    from { opacity: 0; transform: translate(-50%, -10px); }
    to { opacity: 1; transform: translate(-50%, 0); }
  }

  .status-badge {
    display: inline-flex;
    align-items: center;
    padding: 3px 10px;
    border-radius: 10px;
    color: white;
    font-size: 12px;
    text-align: center;
    min-width: 80px;
    gap: 5px;
  }

  .status-badge.checkin {
    background-color: #4caf50;
  }

  .status-badge.checkout {
    background-color: #2196f3;
  }

  .status-badge.absent {
    background-color: #f44336;
  }

  .status-badge.late {
    background-color: #ff9800;
  }

  .action-btn {
    width: 30px;
    height: 30px;
    border-radius: 50%;
    display: inline-flex;
    justify-content: center;
    align-items: center;
    cursor: pointer;
    color: white;
  }

  .view-btn {
    background-color: #2196f3;
  }

  .delete-btn {
    background-color: #f44336;
  }

  .pagination-container {
    background-color: #15273a;
    padding: 15px;
    border-radius: 5px;
    display: flex;
    justify-content: space-between;
    align-items: center;
    margin-top: 20px;
    gap: 5px;
  }

  .pagination {
    display: flex;
    justify-content: center;
    margin-top: 20px;
    gap: 5px;
  }

  .pagination button {
    padding: 8px 12px;
    background-color: #234666;
    color: white;
    border: none;
    border-radius: 5px;
    cursor: pointer;
    transition: all 0.3s ease;
  }

  .pagination button:hover {
    background-color: #1a3550;
  }

  .pagination button.active {
    background-color: #4caf50;
  }

  .add-record-btn {
    background-color: #4caf50;
    color: white;
    border: none;
    padding: 8px 15px;
    border-radius: 3px;
    cursor: pointer;
    display: flex;
    align-items: center;
    gap: 5px;
  }

  .no-records {
    text-align: center;
    padding: 20px;
    font-style: italic;
    color: #777;
  }
  </style>