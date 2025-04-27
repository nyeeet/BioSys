    <script>
    import AccountHeader from '../../components/AccountHeader.vue'
    import Sidebar from '../../components/Sidebar.vue'
    import Chart from 'chart.js/auto'

    export default {
      components: {
        AccountHeader,
        Sidebar,
      },
      data() {
        return {
          isCollapsed: true,
          activeMenu: 'analytics',
          isDatePickerOpen: false,
          startDate: '',
          endDate: '',
          selectedMonth: '',
          currentDate: new Date(),
          totalAttendance: 0, 
          totalPercentage: '0%',
          dateError: '',
        
          activeCategory: '',
          // Attendance records will be fetched and stored here
          attendanceRecords: [],
          // Chart data stored in data properties with percentage values
          chartData: {
            weekdays: ['Monday', 'Tuesday', 'Wednesday', 'Thursday', 'Friday'],
            checkIn: {
              data: [0, 0, 0, 0, 0],
              backgroundColor: [
                'rgba(255, 0, 0, 0.8)',
                'rgba(237, 0, 255, 0.8)',
                'rgba(0, 108, 255, 0.8)',
                'rgba(241, 255, 0, 0.8)',
                'rgba(245, 40, 145, 0.8)',
              ],
              borderColor: 'rgba(75, 192, 192, 1)'
            },
            checkOut: {
              data: [0, 0, 0, 0, 0],
              backgroundColor: 'rgba(118, 171, 255, 0.8)',
              borderColor: 'rgba(31, 115, 251, 0.8)'
            },
            absents: {
              data: [0, 0, 0, 0, 0],
              backgroundColor: [
                'rgba(255, 0, 0, 0.8)',
                'rgba(237, 0, 255, 0.8)',
                'rgba(0, 108, 255, 0.8)',
                'rgba(241, 255, 0, 0.8)',
                'rgba(245, 40, 145, 0.8)',
              ],
            },
            lates: {
              data: [0, 0, 0, 0, 0],
              backgroundColor: [
                'rgba(255, 0, 0, 0.8)',
                'rgba(237, 0, 255, 0.8)',
                'rgba(0, 108, 255, 0.8)',
                'rgba(241, 255, 0, 0.8)',
                'rgba(245, 40, 145, 0.8)',
              ]
            }
          },
          charts: {
            polarChart: null,
            lineChart: null,
            barChart: null,
            doughnutChart: null
          },
          // Add daily totals to calculate percentages
          dailyTotals: [0, 0, 0, 0, 0]
        }
      },
      computed: {
        formattedDate() {
          const options = { weekday: 'long', year: 'numeric', month: 'long', day: 'numeric' };
          return this.currentDate.toLocaleDateString('en-US', options);
        },
        maxDate() {
          // Format current date as YYYY-MM-DD for input max attribute
          const year = this.currentDate.getFullYear();
          const month = String(this.currentDate.getMonth() + 1).padStart(2, '0');
          const day = String(this.currentDate.getDate()).padStart(2, '0');
          return `${year}-${month}-${day}`;
        },
        isDateRangeValid() {
          if (!this.startDate || !this.endDate) return false; // Changed to false to require both dates
          
          const startDateObj = new Date(this.startDate);
          const endDateObj = new Date(this.endDate);
          const currentDateObj = new Date(this.currentDate);
          
          // Set hours to 0 for proper comparison
          startDateObj.setHours(0, 0, 0, 0);
          endDateObj.setHours(0, 0, 0, 0);
          currentDateObj.setHours(0, 0, 0, 0);
          
          return startDateObj <= endDateObj && endDateObj <= currentDateObj;
        }
      },
      mounted() {
        // Check if there's a category in the route query
        if (this.$route.query.category) {
          this.activeCategory = this.$route.query.category;
        }
        
        // Fetch attendance records first then initialize charts
        this.fetchAttendanceRecords();
      },
      watch: {
        // Watch for changes in dates to validate them
        startDate() {
          this.validateDateRange();
        },
        endDate() {
          this.validateDateRange();
        }
      },
      methods: {
        validateDateRange() {
          this.dateError = '';
          
          if (!this.startDate || !this.endDate) {
            this.dateError = 'Please select both start and end dates';
            return;
          }
          
          const startDateObj = new Date(this.startDate);
          const endDateObj = new Date(this.endDate);
          const currentDateObj = new Date(this.currentDate);
          
          // Set hours to 0 for proper comparison
          startDateObj.setHours(0, 0, 0, 0);
          endDateObj.setHours(0, 0, 0, 0);
          currentDateObj.setHours(0, 0, 0, 0);
          
          if (startDateObj > endDateObj) {
            this.dateError = 'Start date cannot be after end date';
          } else if (endDateObj > currentDateObj) {
            this.dateError = 'Cannot select future dates';
          }
        },
        openDatePicker() {
          this.isDatePickerOpen = true;
          // Clear previous values
          this.dateError = '';
        },
        closeDatePicker() {
          this.isDatePickerOpen = false;
          this.dateError = '';
        },
        forceApplyDateFilter() {
          // Force apply the filter even if validation fails
          let dateRange = '';
          
          if (this.selectedMonth) {
            const year = new Date().getFullYear();
            const monthName = new Date(year, parseInt(this.selectedMonth) - 1, 1).toLocaleString('default', { month: 'long' });
            dateRange = `${monthName} ${year}`;
        
            this.fetchAttendanceRecordsForMonth(this.selectedMonth);
          } else if (this.startDate && this.endDate) {
            const start = new Date(this.startDate).toLocaleDateString();
            const end = new Date(this.endDate).toLocaleDateString();
            dateRange = `${start} to ${end}`;
            
            this.fetchAttendanceRecordsForDateRange(this.startDate, this.endDate);
          }
          
          if (dateRange) {
            console.log(`Applying date filter: ${dateRange}`);
          }
          
          this.isDatePickerOpen = false;
        },
        applyDateFilter() {
          if (this.startDate && this.endDate) {
            this.forceApplyDateFilter();
          } else {
            this.dateError = 'Please select both start and end dates';
          }
        },
        
        scrollToCategory(category) {
          // Find the chart element based on category
          const chartIds = {
            'check-in': 'checkInChart',
            'check-out': 'checkOutChart', 
            'absents': 'absentsChart',
            'lates': 'latesChart'
          };
          
          const chartId = chartIds[category];
          if (chartId) {
            const chartElement = document.getElementById(chartId);
            if (chartElement) {
              // Scroll to the chart with a smooth animation
              chartElement.scrollIntoView({ 
                behavior: 'smooth', 
                block: 'center' 
              });
              
              // Add a highlight effect to the chart
              chartElement.classList.add('highlight-chart');
              setTimeout(() => {
                chartElement.classList.remove('highlight-chart');
              }, 2000);
            }
          }
        },

        // New method to fetch attendance records
        fetchAttendanceRecords() {
          console.log('Fetching attendance records...');
          
          this.attendanceRecords = this.generateMockAttendanceRecords();
          
          // Process the records to calculate percentages
          this.processAttendanceData();
          
          // Initialize charts after processing data
          this.initCharts();
          
          // Scroll to category if needed
          if (this.activeCategory) {
            this.$nextTick(() => {
              this.scrollToCategory(this.activeCategory);
            });
          }
        },
        
        fetchAttendanceRecordsForMonth(month) {
          console.log(`Fetching attendance records for month: ${month}`);
          
          this.attendanceRecords = this.generateMockAttendanceRecords(month);
          
          // Process the records to calculate percentages
          this.processAttendanceData();
          
          // Update charts with new data
          this.updateCharts();
        },
        
        fetchAttendanceRecordsForDateRange(startDate, endDate) {
          console.log(`Fetching attendance records from ${startDate} to ${endDate}`);
          
          this.attendanceRecords = this.generateMockAttendanceRecords(null, startDate, endDate);
          
          // Process the records to calculate percentages
          this.processAttendanceData();
          
          // Update charts with new data
          this.updateCharts();
        },
        
        // Generate mock attendance records for testing
        generateMockAttendanceRecords(month = null, startDate = null, endDate = null) {
          const records = [];
          const statuses = [
            'checkin_am', 'checkin_pm', 'checkout_am', 'checkout_pm',
            'absent_am', 'absent_pm', 'late_am', 'late_pm'
          ];
          
          // Generate between 100-200 records
          const totalRecords = Math.floor(Math.random() * 100) + 100;
          
          for (let i = 0; i < totalRecords; i++) {
            // Create a random date within the last 30 days
            const date = new Date();
            date.setDate(date.getDate() - Math.floor(Math.random() * 30));
            
            // Apply month filter if specified
            if (month !== null) {
              date.setMonth(parseInt(month) - 1);
            }
            
            // Apply date range filter if specified
            if (startDate && endDate) {
              const start = new Date(startDate);
              const end = new Date(endDate);
              
              // Generate a random date between start and end
              const timeRange = end.getTime() - start.getTime();
              date.setTime(start.getTime() + Math.random() * timeRange);
            }
            
            // Get day of week (0 = Sunday, 1 = Monday, ..., 6 = Saturday)
            const dayOfWeek = date.getDay();
            
            // Skip weekends (Saturday and Sunday)
            if (dayOfWeek === 0 || dayOfWeek === 6) continue;
            
            // Convert to weekday index (0 = Monday, ..., 4 = Friday)
            const weekdayIndex = dayOfWeek === 0 ? 6 : dayOfWeek - 1;
            
            records.push({
              id: `record-${i}`,
              date: date.toISOString().split('T')[0],
              weekdayIndex: Math.min(weekdayIndex, 4), // Ensure index is between 0-4
              status: statuses[Math.floor(Math.random() * statuses.length)]
            });
          }
          
          return records;
        },
        
        // Process attendance records to calculate percentages
        processAttendanceData() {
          // Reset chart data and daily totals
          this.chartData.checkIn.data = [0, 0, 0, 0, 0];
          this.chartData.checkOut.data = [0, 0, 0, 0, 0];
          this.chartData.absents.data = [0, 0, 0, 0, 0];
          this.chartData.lates.data = [0, 0, 0, 0, 0];
          this.dailyTotals = [0, 0, 0, 0, 0];
          
          // Track total counters per weekday
          const counters = {
            checkIn: [0, 0, 0, 0, 0],
            checkOut: [0, 0, 0, 0, 0],
            absents: [0, 0, 0, 0, 0],
            lates: [0, 0, 0, 0, 0]
          };
          
          // Process each record
          this.attendanceRecords.forEach(record => {
            const index = record.weekdayIndex;
            
            // Only process valid weekday indices (0-4)
            if (index >= 0 && index <= 4) {
              // Increment daily total for this weekday
              this.dailyTotals[index]++;
              
              // Increment the appropriate counter based on status
              if (record.status.includes('checkin')) {
                counters.checkIn[index]++;
              } else if (record.status.includes('checkout')) {
                counters.checkOut[index]++;
              } else if (record.status.includes('absent')) {
                counters.absents[index]++;
              } else if (record.status.includes('late')) {
                counters.lates[index]++;
              }
            }
          });
          
          // Calculate percentages for each category and day
          for (let i = 0; i < 5; i++) {
            if (this.dailyTotals[i] > 0) {
              this.chartData.checkIn.data[i] = Math.round((counters.checkIn[i] / this.dailyTotals[i]) * 100);
              this.chartData.checkOut.data[i] = Math.round((counters.checkOut[i] / this.dailyTotals[i]) * 100);
              this.chartData.absents.data[i] = Math.round((counters.absents[i] / this.dailyTotals[i]) * 100);
              this.chartData.lates.data[i] = Math.round((counters.lates[i] / this.dailyTotals[i]) * 100);
            }
          }
          
          // Calculate overall attendance percentage
          const totalCheckIns = counters.checkIn.reduce((sum, val) => sum + val, 0);
          const totalRecords = this.dailyTotals.reduce((sum, val) => sum + val, 0);
          
          if (totalRecords > 0) {
            this.totalAttendance = Math.round((totalCheckIns / totalRecords) * 100);
            this.totalPercentage = `${this.totalAttendance}%`;
          }
        },
      
        updateCharts() {
          if (this.charts.polarChart) {
            this.charts.polarChart.data.datasets[0].data = this.chartData.checkIn.data;
            this.charts.polarChart.update();
          }
          
          if (this.charts.lineChart) {
            this.charts.lineChart.data.datasets[0].data = this.chartData.checkOut.data;
            this.charts.lineChart.update();
          }
          
          if (this.charts.barChart) {
            this.charts.barChart.data.datasets[0].data = this.chartData.absents.data;
            this.charts.barChart.update();
          }
          
          if (this.charts.doughnutChart) {
            this.charts.doughnutChart.data.datasets[0].data = this.chartData.lates.data;
            this.charts.doughnutChart.update();
          }
        },
        
        initCharts() {
          this.$nextTick(() => {
            // doughnut chart for lates
            const doughnutCtx = document.getElementById('doughnutChart').getContext('2d');
            this.charts.doughnutChart = new Chart(doughnutCtx, {
              type: 'doughnut',
              data: {
                labels: this.chartData.weekdays,
                datasets: [{
                  data: this.chartData.lates.data,
                  backgroundColor: this.chartData.lates.backgroundColor
                }]
              },
              options: {
                responsive: true,
                maintainAspectRatio: false,
                plugins: {
                  tooltip: {
                    callbacks: {
                      label: function(context) {
                        return `${context.label}: ${context.raw}%`;
                      }
                    }
                  }
                }
              }
            });
            
            // polar chart for check-in
            const polarCtx = document.getElementById('polarChart').getContext('2d');
            this.charts.polarChart = new Chart(polarCtx, {
              type: 'polarArea',
              data: {
                labels: this.chartData.weekdays,
                datasets: [{
                  data: this.chartData.checkIn.data,
                  backgroundColor: this.chartData.checkIn.backgroundColor,
                  borderColor: this.chartData.checkIn.borderColor,
                  borderWidth: 1
                }]
              },
              options: {
                responsive: true,
                maintainAspectRatio: false,
                scales: {
                  r: {
                    ticks: {
                      display: false
                    }
                  }
                },
                plugins: {
                  tooltip: {
                    callbacks: {
                      label: function(context) {
                        return `${context.label}: ${context.raw}%`;
                      }
                    }
                  }
                }
              }
            });
            
          
            const lineCtx = document.getElementById('lineChart').getContext('2d');
            this.charts.lineChart = new Chart(lineCtx, {
              type: 'line',
              data: {
                labels: this.chartData.weekdays,
                datasets: [{
                  data: this.chartData.checkOut.data,
                  backgroundColor: this.chartData.checkOut.backgroundColor,
                  borderColor: this.chartData.checkOut.borderColor,
                  tension: 0.4,
                  fill: true
                }]
              },
              options: {
                responsive: true,
                maintainAspectRatio: false,
                plugins: {
                  tooltip: {
                    callbacks: {
                      label: function(context) {
                        return `${context.label}: ${context.raw}%`;
                      }
                    }
                  }
                },
                scales: {
                  y: {
                    beginAtZero: true,
                    max: 100,
                    ticks: {
                      callback: function(value) {
                        return value + '%';
                      }
                    }
                  }
                }
              }
            });
          
            const barCtx = document.getElementById('barChart').getContext('2d');
            this.charts.barChart = new Chart(barCtx, {
              type: 'bar',
              data: {
                labels: this.chartData.weekdays,
                datasets: [{
                  data: this.chartData.absents.data,
                  backgroundColor: this.chartData.absents.backgroundColor
                }]
              },
              options: {
                responsive: true,
                maintainAspectRatio: false,
                plugins: {
                  tooltip: {
                    callbacks: {
                      label: function(context) {
                        return `${context.label}: ${context.raw}%`;
                      }
                    }
                  }
                },
                scales: {
                  y: {
                    beginAtZero: true,
                    max: 100,
                    ticks: {
                      callback: function(value) {
                        return value + '%';
                      }
                    }
                  }
                }
              }
            });
          });
        }
      }
    }
    </script>

    <template>
      <link href="https://fonts.googleapis.com/css2?family=Poppins:wght@300;400;500;600;700&display=swap" 
        rel="stylesheet" />

      <link rel="stylesheet" 
        href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.4.0/css/all.min.css" />

      <aside>
        <Sidebar />
      </aside>

      <div class="main-content">
        <div class="dashboard-container">
          <AccountHeader />
          <div class="header-actions">
            <h1>ANALYTICS</h1>
            
            <div class="date-display">
              <p class="date-highlight">{{ formattedDate }}</p>
          
              <div class="charts-navigation">
                <div class="navigation-controls">
                  <div class="header-nav-item" @click="openDatePicker">
                    <i class="fas fa-calendar"></i>
                  </div>
                  <div class="header-nav-item">
                    <i class="fas fa-download"></i>
                  </div>
                </div>
              </div>
            </div>

            <div class="charts-container">
              <div id="checkInChart" class="chart-card" :class="{'active-chart': activeCategory === 'check-in'}">
                <h2 class="chart-title">Check-In Percentage</h2>
                <div class="chart-description">
                  Daily check-in percentage statistics tracked throughout the week
                </div>
                <div class="polar-chart-container">
                  <canvas id="polarChart"></canvas>
                </div>
              </div>
              
              <div id="checkOutChart" class="chart-card" :class="{'active-chart': activeCategory === 'check-out'}">
                <h2 class="chart-title">Check-Out Percentage</h2>
                <div class="chart-description">
                  Daily check-out percentage statistics tracked throughout the week
                </div>
                <div class="polar-chart-container">
                  <canvas id="lineChart"></canvas>
                </div>
              </div>
              
              <div id="absentsChart" class="chart-card" :class="{'active-chart': activeCategory === 'absents'}">
                <h2 class="chart-title">Absence Percentage</h2>
                <div class="chart-description">
                  Student absence percentage throughout the week
                </div>
                <div class="polar-chart-container">
                  <canvas id="barChart"></canvas>
                </div>
              </div>
              
              <div id="latesChart" class="chart-card" :class="{'active-chart': activeCategory === 'lates'}">
                <h2 class="chart-title">Late Arrivals Percentage</h2>
                <div class="chart-description">
                  Students arriving late percentage throughout the week
                </div>
                <div class="polar-chart-container">
                  <canvas id="doughnutChart"></canvas>
                </div>
              </div>
            </div>
          </div>
          
          <div class="date-picker-modal" :class="{ active: isDatePickerOpen }">
            <div class="date-picker-container">
              <div class="date-picker-header">
                <div class="date-picker-title">Select Date Range</div>
                <div class="date-picker-close" @click="closeDatePicker">×</div>
              </div>
              
              <div class="date-picker-body">
                <div class="date-range-container">
                  <div class="date-input-group">
                    <label class="date-input-label">Start Date</label>
                    <input type="date" class="date-input" v-model="startDate" :max="maxDate">
                  </div>
                  
                  <div class="date-input-group">
                    <label class="date-input-label">End Date</label>
                    <input type="date" class="date-input" v-model="endDate" :max="maxDate">
                  </div>
                </div>
                
                <div v-if="dateError" class="date-error-message">
                  {{ dateError }}
                </div>
              </div>
              
              <div class="date-picker-footer">
                <button class="date-picker-button cancel-button" @click="closeDatePicker">Cancel</button>
                <!-- Removed disabled attribute to ensure button is always clickable -->
                <button class="date-picker-button apply-button" @click="applyDateFilter">OK</button>
              </div>
            </div>
          </div>
        </div>
      </div>
    </template>


    <style scoped>
    h1 {
      text-align: center;
      margin-bottom: 10px;
      font-size: 32px;
      color: #ddd;
    }

    .main-content {
      display: flex;
      flex-direction: column;
      margin-left: 6%;
      width: 93%;
    }

    .main-content.expanded {
      margin-left: 250px;
    }

    .date-display {
      margin-bottom: 30px;
    }

    .date-highlight {
      font-size: 18px;
      color: #34495e;
      background-color: #f8f9fa;
      padding: 8px 20px;
      border-radius: 20px;
      display: inline-block;
      box-shadow: 0 2px 5px rgba(0,0,0,0.1);
      border-left: 4px solid;
      font-weight: 500;
    }

    .navigation {
      display: flex;
      gap: 15px;
    }

    .navigation-controls {
      display: flex;
      gap: 20px;
      margin-left: 15px;
      margin-bottom: 20px;
    }
      
    .header-nav-item {
      width: 40px;
      height: 40px;
      border: 2px solid #34495e;
      display: flex;
      align-items: center;
      justify-content: center;
      cursor: pointer;
      border-radius: 5px;
      transition: all 0.3s;
    }
            
    .header-nav-item:hover {
      background-color: #34495e;
      color: white;
    }

    .charts-container {
      display: grid;
      grid-template-columns: 1fr 1fr;
      gap: 30px;
      margin-top: 30px;
    }

    .chart-card {
      background-color: white;
      border-radius: 10px;
      box-shadow: 0 4px 12px rgba(0, 0, 0, 0.1);
      padding: 20px;
      position: relative;
    }
            
    .chart-title {
      font-size: 18px;
      margin-bottom: 20px;
      color: #2c3e50;
      font-weight: 600;
    }
            
    .polar-chart-container {
      height: 350px;
      position: relative;
    }
            
    .polar-chart-label {
      position: absolute;
      right: 20px;
      bottom: 40px;
      font-size: 14px;
      color: #7f8c8d;
    }
            
    /* Date Picker Modal Styles */
    .date-picker-modal {
      position: fixed;
      top: 0;
      left: 0;
      width: 100%;
      height: 100%;
      background-color: rgba(0, 0, 0, 0.5);
      display: flex;
      justify-content: center;
      align-items: center;
      z-index: 100;
      opacity: 0;
      pointer-events: none;
      transition: opacity 0.3s ease;
    }
            
    .date-picker-modal.active {
      opacity: 1;
      pointer-events: auto;
    }
            
    .date-picker-container {
      background-color: white;
      border-radius: 10px;
      padding: 25px;
      width: 90%;
      max-width: 400px;
      box-shadow: 0 5px 20px rgba(0, 0, 0, 0.2);
      position: relative;
    }
            
    .date-picker-header {
      display: flex;
      justify-content: space-between;
      align-items: center;
      margin-bottom: 20px;
    }
            
    .date-picker-title {
      font-size: 18px;
      font-weight: 600;
      color: #2c3e50;
    }
            
    .date-picker-close {
      cursor: pointer;
      font-size: 20px;
      color: #7f8c8d;
    }
            
    .date-picker-body {
      margin-bottom: 20px;
    }
            
    .date-range-container {
      display: flex;
      gap: 10px;
      margin-bottom: 20px;
    }
            
    .date-input-group {
      flex: 1;
    }
            
    .date-input-label {
      display: block;
      font-size: 14px;
      margin-bottom: 5px;
      color: #7f8c8d;
    }
            
    .date-input {
      width: 100%;
      padding: 10px;
      border: 1px solid #ddd;
      border-radius: 5px;
      font-size: 14px;
    }
            
    .month-selector {
      margin-top: 15px;
    }
            
    .month-selector select {
      width: 100%;
      padding: 10px;
      border: 1px solid #ddd;
      border-radius: 5px;
      font-size: 14px;
    }
            
    .date-picker-footer {
      display: flex;
      justify-content: flex-end;
      gap: 10px;
    }
            
    .date-picker-button {
      padding: 10px 15px;
      border: none;
      border-radius: 5px;
      cursor: pointer;
      font-weight: 500;
      transition: all 0.3s ease;
    }
            
    .cancel-button {
      background-color: #e0e0e0;
      color: #333;
    }
            
    .apply-button {
      background-color: #4caf50;
      color: white;
    }
            
    .cancel-button:hover {
      background-color: #d5d5d5;
    }
            
    .apply-button:hover {
      background-color: #43a047;
    }


    .active-chart {
      border: 2px solid #3498db;
      box-shadow: 0 0 15px rgba(52, 152, 219, 0.3);
    }

    .chart-description {
      color: #7f8c8d;
      font-size: 14px;
      margin-bottom: 20px;
    }
    </style>