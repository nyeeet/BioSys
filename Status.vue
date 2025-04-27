<script>
export default {
  data() {
    return {
      currentDate: new Date(),
      analyticsData: {
        checkIn: 0,
        checkOut: 0,
        absents: 0,
        lates: 0
      },
      trends: {
        checkIn: { value: 0, period: 'last month', isUp: true },
        checkOut: { value: 0, period: 'last week', isUp: true },
        absents: { value: 0, period: 'last month', isUp: true },
        lates: { value: 0, period: 'yesterday', isUp: false }
      }
    }
  },
  computed: {
    formattedDate() {
      const options = { weekday: 'long', year: 'numeric', month: 'long', day: 'numeric' };
      return this.currentDate.toLocaleDateString('en-US', options);
    },
    absentPercentage() {
      // Calculate absent percentage based on total potential attendees
      const totalStudents = 100; // This should come from your data source
      return this.analyticsData.absents ? `${Math.round((this.analyticsData.absents / totalStudents) * 100)}%` : '0%';
    }
  },
  mounted() {
    // Fetch analytics data when component is mounted
    this.fetchAnalyticsData();
  },
  methods: {
    navigateToAnalytics(category) {
      this.$router.push({
        path: '/analytics',
        query: { category: category }
      });
    },
    fetchAnalyticsData() {
      // In a real application, this would be an API call
      // For now, we'll calculate totals from the chart data
      
      // Get chart data from parent component or vuex store
      // This is just a placeholder - implement actual data fetching logic
      const chartData = this.getChartData();
      
      if (chartData) {
        // Calculate today's values (use the latest data point or sum as needed)
        this.analyticsData.checkIn = this.sumArray(chartData.checkIn.data);
        this.analyticsData.checkOut = this.sumArray(chartData.checkOut.data);
        this.analyticsData.absents = this.sumArray(chartData.absents.data) / 5; // Average per day
        this.analyticsData.lates = this.sumArray(chartData.lates.data);
        
        // You would also fetch trend data in a real application
        // For now we'll keep the existing trend percentages
      }
    },
    getChartData() {
      // In a real application, you would:
      // 1. Get data from Vuex store, or
      // 2. Get data via props from parent component, or
      // 3. Make an API call directly
      
      // For demo purposes, return sample data structure similar to what's in the Analytics component
      return {
        weekdays: ['Monday', 'Tuesday', 'Wednesday', 'Thursday', 'Friday'],
        checkIn: { data: [31, 36, 27, 34, 24] },
        checkOut: { data: [12, 19, 3, 5, 2] },
        absents: { data: [5, 9, 8, 11, 6] },
        lates: { data: [2, 1, 3, 0, 1] }
      };
    },
    sumArray(arr) {
      if (!arr || !Array.isArray(arr)) return 0;
      return arr.reduce((sum, val) => sum + val, 0);
    }
  }
}
</script>

<template>
    <div class="stats-container">
        <div class="stat-card" @click="navigateToAnalytics('check-in')">
            <div class="stat-header">
            <div>
                <h3 class="stat-value">{{ analyticsData.checkIn }}</h3>
                <p class="stat-label">Total Check-ins Today</p>
            </div>
            <div class="stat-icon">
                <i class="fas fa-user-check"></i> 
            </div>
            </div>
            <div class="stat-trend">
              <i :class="trends.checkIn.isUp ? 'fas fa-arrow-up' : 'fas fa-arrow-down'"></i> 
              {{ trends.checkIn.value }}% from {{ trends.checkIn.period }}
            </div>
        </div>

        <div class="stat-card" @click="navigateToAnalytics('check-out')">
            <div class="stat-header">
            <div>
                <h3 class="stat-value">{{ analyticsData.checkOut }}</h3>
                <p class="stat-label">Total Check-outs Today</p>
            </div>
            <div
                class="stat-icon"
                style="background-color: rgba(33, 150, 243, 0.2); color: #2196f3"
            >
                <i class="fas fa-user-times"></i>
            </div>
            </div>
            <div class="stat-trend">
              <i :class="trends.checkOut.isUp ? 'fas fa-arrow-up' : 'fas fa-arrow-down'"></i>
              {{ trends.checkOut.value }}% from {{ trends.checkOut.period }}
            </div>
        </div>

        <div class="stat-card" @click="navigateToAnalytics('absents')">
            <div class="stat-header">
            <div>
                <h3 class="stat-value">{{ absentPercentage }}</h3>
                <p class="stat-label">Absent Students</p>
            </div>
            <div class="stat-icon" style="background-color: rgba(255, 152, 0, 0.2); color: #ff9800">
                <i class="fas fa-user-slash"></i>
            </div>
            </div>
            <div class="stat-trend">
              <i :class="trends.absents.isUp ? 'fas fa-arrow-up' : 'fas fa-arrow-down'"></i>
              {{ trends.absents.value }}% from {{ trends.absents.period }}
            </div>
        </div>

        <div class="stat-card" @click="navigateToAnalytics('lates')">
            <div class="stat-header">
            <div>
                <h3 class="stat-value">{{ analyticsData.lates }}</h3>
                <p class="stat-label">Late Check-ins</p>
            </div>
            <div class="stat-icon" style="background-color: rgba(244, 67, 54, 0.2); color: #f44336">
                <i class="fas fa-hourglass-half"></i>
            </div>
            </div>
            <div class="stat-trend" :class="{'trend-down': !trends.lates.isUp}">
              <i :class="trends.lates.isUp ? 'fas fa-arrow-up' : 'fas fa-arrow-down'"></i>
              {{ trends.lates.value }}% from {{ trends.lates.period }}
            </div>
        </div>
    </div>
</template>

<style scoped>
.stats-container {
    display: grid;
    grid-template-columns: repeat(4, 1fr);
    gap: 18px;
    margin-bottom: 30px;
}
.stat-card {
    background-color: #1a3550;
    border-radius: 10px;
    padding: 20px;
    display: flex;
    flex-direction: column;
    box-shadow: 0 2px 10px rgba(0, 0, 0, 0.2);
    transition: transform 0.3s ease;
}
.stat-card:hover {
    transform: translateY(-5px);
    cursor: pointer;
}
.stat-header {
    display: flex;
    justify-content: space-between;
    align-items: center;
    margin-bottom: 15px;
}
.stat-icon {
    width: 50px;
    height: 50px;
    background-color: rgba(76, 175, 80, 0.2);
    border-radius: 10px;
    display: flex;
    align-items: center;
    justify-content: center;
    color: #4caf50;
    font-size: 24px;
}
.stat-value {
    font-size: 28px;
    font-weight: 600;
    margin: 0;
}
.stat-label {
    font-size: 14px;
    color: #b3b3b3;
    margin: 0;
}
.stat-trend {
    display: flex;
    align-items: center;
    font-size: 14px;
    color: #4caf50;
    margin-top: 5px;
}
.status-badge {
    padding: 5px 10px;
    border-radius: 12px;
    font-size: 12px;
    font-weight: 500;
    text-align: center;
    display: inline-block;
}
.status-upcoming {
    background-color: rgba(33, 150, 243, 0.2);
    color: #2196f3;
    border: 1px solid #2196f3;
}
.status-cancelled {
    background-color: rgba(244, 67, 54, 0.2);
    color: #f44336;
    border: 1px solid #f44336;
}
.status-ongoing {
  background-color: rgba(255, 193, 7, 0.2);
    color: #ffc107;
    border: 1px solid #ffc107;
}
.status-completed {
    background-color: rgba(76, 175, 80, 0.2);
    color: #4caf50;
    border: 1px solid #4caf50;
}
</style>