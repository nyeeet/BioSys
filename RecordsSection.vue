  <script setup>
    import AccountHeader from '../../components/AccountHeader.vue'
    import Sidebar from '../../components/Sidebar.vue'
    import ViewModal from '../../components/ViewModal.vue'
    import DeleteModal from '../../components/DeleteModal.vue'
    import { ref, computed, onMounted, watch } from 'vue'
    import { useRouter } from 'vue-router'

    const router = useRouter()

    // Grade Levels data
    const gradeLevels = ref([])
    const currentPage = ref(1)
    const itemsPerPage = ref(5)
    const selectedGradeLevel = ref(null)
    const showViewModal = ref(false)
    const showDeleteModal = ref(false)
    const searchFilter = ref('')
    const sortField = ref('id')
    const sortDirection = ref('asc')
    const sortOption = ref('id-asc')

    // Load grade levels from localStorage on component mount
    onMounted(() => {
      loadGradeLevels()
    })

    // Function to load grade levels from localStorage
    function loadGradeLevels() {
      const savedGradeLevels = localStorage.getItem('gradeLevels')
      if (savedGradeLevels) {
        gradeLevels.value = JSON.parse(savedGradeLevels)
      } else {
        gradeLevels.value = []
        localStorage.setItem('gradeLevels', JSON.stringify(gradeLevels.value))
      }
    }

    // Sort grade levels
    const sortedGradeLevels = computed(() => {
      const [field, direction] = sortOption.value.split('-')
      sortField.value = field
      sortDirection.value = direction
      
      return [...gradeLevels.value].sort((a, b) => {
        let compareA, compareB
        
        if (field === 'id') {
          compareA = a.id
          compareB = b.id
        } else if (field === 'projectName') {
          compareA = a.projectName || ''
          compareB = b.projectName || ''
        } else if (field === 'section') {
          compareA = a.section || ''
          compareB = b.section || ''
        }

        if (typeof compareA === 'string' && typeof compareB === 'string') {
          if (direction === 'asc') {
            return compareA.localeCompare(compareB)
          } else {
            return compareB.localeCompare(compareA)
          }
        }
        
        if (direction === 'asc') {
          return compareA - compareB
        } else {
          return compareB - compareA
        }
      })
    })

    // Filter grade levels based on search
    const filteredGradeLevels = computed(() => {
      if (!searchFilter.value) return sortedGradeLevels.value
      
      const query = searchFilter.value.toLowerCase()
      return sortedGradeLevels.value.filter(grade => 
        grade.projectName.toLowerCase().includes(query) ||
        grade.section.toLowerCase().includes(query) ||
        grade.id.includes(query)
      )
    })

    // Paginate grade levels
    const paginatedGradeLevels = computed(() => {
      const start = (currentPage.value - 1) * itemsPerPage.value
      const end = start + itemsPerPage.value
      return filteredGradeLevels.value.slice(start, end)
    })

    // Calculate total pages
    const totalPages = computed(() => {
      return Math.ceil(filteredGradeLevels.value.length / itemsPerPage.value)
    })

    watch(sortOption, () => {
      currentPage.value = 1
    })

    watch(searchFilter, () => {
      currentPage.value = 1
    })

    function changeSort(field) {
      if (sortField.value === field) {
        const newDirection = sortDirection.value === 'asc' ? 'desc' : 'asc'
        sortOption.value = `${field}-${newDirection}`
      } else {
        sortOption.value = `${field}-asc`
      }
    }

    function openViewModal(gradeLevel) {
      selectedGradeLevel.value = gradeLevel
      showViewModal.value = true
    }

    function openDeleteModal(gradeLevel) {
      selectedGradeLevel.value = gradeLevel
      showDeleteModal.value = true
    }

    function closeViewModal() {
      showViewModal.value = false
    }

    function closeDeleteModal() {
      showDeleteModal.value = false
    }

    function deleteGradeLevel(gradeLevelId) {
      const index = gradeLevels.value.findIndex(g => g.id === gradeLevelId)
      if (index !== -1) {
        gradeLevels.value.splice(index, 1)
        localStorage.setItem('gradeLevels', JSON.stringify(gradeLevels.value))
        
        // Also delete any attendance records associated with this grade level
        const attendanceRecords = JSON.parse(localStorage.getItem('attendanceRecords') || '[]')
        const updatedRecords = attendanceRecords.filter(record => 
          record.sectionId !== gradeLevelId && 
          record.section !== gradeLevels.value[index].section
        )
        localStorage.setItem('attendanceRecords', JSON.stringify(updatedRecords))
      }
    }

    // New function to navigate to record view in new tab
    function openRecordView(gradeLevel) {
      // Encode the section data as a URL parameter
      const sectionData = encodeURIComponent(JSON.stringify(gradeLevel))
      
      // Create the URL with the section ID and data
      const url = `/record-view/${gradeLevel.id}?sectionData=${sectionData}`
      
      // Open in a new tab
      window.open(url, '_blank')
    }

    function goToPage(page) {
      if (page >= 1 && page <= totalPages.value) {
        currentPage.value = page
      }
    }

    function goToFirstPage() {
      currentPage.value = 1
    }

    function goToLastPage() {
      currentPage.value = totalPages.value
    }

    function goToPrevPage() {
      if (currentPage.value > 1) {
        currentPage.value--
      }
    }

    function goToNextPage() {
      if (currentPage.value < totalPages.value) {
        currentPage.value++
      }
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

      <aside>
        <Sidebar />
      </aside>

      <div class="main-content">
        <div class="dashboard-container">
          <AccountHeader />
          <div class="header-actions">
            <h1>RECORDS</h1>

            <div class="table-container">
              <div class="table-actions">
                <div class="search-container">
                  <input 
                    type="text" 
                    v-model="searchFilter" 
                    placeholder="Search by grade level or section..." 
                    class="search-input"
                  />
                </div>
                <div class="sort-container">
                  <label for="sort-select">Sort by:</label>
                  <select id="sort-select" v-model="sortOption" class="sort-select">
                    <option value="id-asc">ID (Low-High)</option>
                    <option value="id-desc">ID (High-Low)</option>
                    <option value="projectName-asc">Grade Level (A-Z)</option>
                    <option value="projectName-desc">Grade Level (Z-A)</option>
                    <option value="section-asc">Section (A-Z)</option>
                    <option value="section-desc">Section (Z-A)</option>
                  </select>
                </div>
              </div>

              <table>
                <thead>
                  <tr>
                    <th>ID</th>
                    <th>GRADE LEVEL</th>
                    <th>SECTION</th>
                    <th>ACTIONS</th>
                  </tr>
                </thead>
                <tbody>
                  <tr 
                    v-for="gradeLevel in paginatedGradeLevels" 
                    :key="gradeLevel.id" 
                    @click="openRecordView(gradeLevel)"
                    class="table-row"
                  >
                    <td>{{ gradeLevel.id }}</td>
                    <td>{{ gradeLevel.projectName }}</td>
                    <td>{{ gradeLevel.section }}</td>
                    <td @click.stop>
                      <div class="action-buttons">
                        <button class="action-button view-button" @click="openViewModal(gradeLevel)">
                          <i class="fas fa-eye"></i>
                        </button>
                        <button class="action-button delete-button" @click="openDeleteModal(gradeLevel)">
                          <i class="fas fa-trash"></i>
                        </button>
                      </div>
                    </td>
                  </tr>
                  <tr v-if="paginatedGradeLevels.length === 0">
                    <td colspan="4" class="no-data">No grade levels found. Add a new grade level to get started.</td>
                  </tr>
                </tbody>
              </table>

              <div class="pagination" v-if="filteredGradeLevels.length > itemsPerPage">
                <button @click="goToFirstPage"><i class="fas fa-angle-double-left"></i></button>
                <button @click="goToPrevPage"><i class="fas fa-angle-left"></i></button>
                <button 
                  v-for="page in totalPages" 
                  :key="page" 
                  :class="{ active: page === currentPage }"
                  @click="goToPage(page)"
                >
                  {{ page }}
                </button>
                <button @click="goToNextPage"><i class="fas fa-angle-right"></i></button>
                <button @click="goToLastPage"><i class="fas fa-angle-double-right"></i></button>
              </div>
              
              <ViewModal 
                :show="showViewModal" 
                :project="selectedGradeLevel" 
                @close="closeViewModal" 
              />

              <DeleteModal 
                :show="showDeleteModal" 
                :project="selectedGradeLevel" 
                @close="closeDeleteModal" 
                @delete="deleteGradeLevel" 
              />
            </div>
          </div>
        </div>
      </div>
    </template>

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
    h1 {
        text-align: center;
        font-size: 40px;
    }

    .table-container {
        margin-top: 20px;
        background-color: #1a3550;
        border-radius: 10px;
        padding: 20px;
        box-shadow: 0 2px 10px rgba(0, 0, 0, 0.2);
    }
    table {
        width: 100%;
        border-collapse: collapse;
        margin: auto;
        border-radius: 8px;
        overflow: hidden;
    }
    th, td {
        padding: 12px 15px;
        text-align: left;
        border-bottom: 1px solid #0e2941; 
    }
    th {
        background-color: #234666; 
        color: white;
        font-weight: 500;
        text-transform: uppercase;
        font-size: 14px;
    }
    td {
        background-color: rgba(242, 242, 242, 0.9); 
        color: #0e2941; 
    }
    tr:last-child td {
        border-bottom: none;
    }
    tr:hover td {
        background-color: rgba(242, 242, 242, 1);
    }

    .table-actions {
      display: flex;
      justify-content: space-between;
      margin-bottom: 1rem;
      align-items: center;
    }

    .search-container {
      flex: 1;
      max-width: 300px;
    }

    .search-input {
      width: 100%;
      padding: 0.5rem;
      border: 1px solid #ddd;
      border-radius: 4px;
    }

    .sort-container {
      display: flex;
      align-items: center;
      gap: 0.5rem;
    }

    .sort-select {
      padding: 0.5rem;
      border: 1px solid #ddd;
      border-radius: 4px;
    }


    .sort-indicator {
      margin-left: 0.25rem;
    }


    .action-buttons {
      display: flex;
      gap: 0.5rem;
    }

    .action-button {
      background: none;
      border: none;
      cursor: pointer;
      padding: 0.25rem 0.5rem;
      border-radius: 4px;
    }

    .view-button {
      color: #2196F3;
    }

    .delete-button {
      color: #F44336;
    }

    .no-data {
      text-align: center;
      color: #888;
      padding: 2rem;
    }

    .pagination {
      display: flex;
      justify-content: center;
      margin-top: 1rem;
      gap: 0.25rem;
    }

    .pagination button {
      padding: 0.5rem 0.75rem;
      border: 1px solid #ddd;
      background-color: white;
      cursor: pointer;
      border-radius: 4px;
    }

    .pagination button.active {
      background-color: #2196F3;
      color: white;
      border-color: #2196F3;
    }
    </style>