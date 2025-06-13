<template>
  <el-card class="faction-comparison-card">
    <template #header>
      <div class="card-header">
        <h2>Faction Strength Comparison Analyzer</h2>
      </div>
    </template>
    <p>Enter two faction IDs to analyze and compare their strengths, including RW data, Chain data, and member personal data from the past four months.</p>
    
    <!-- API Key Configuration -->
    <el-card class="api-config-card" style="margin-bottom: 20px;">
      <template #header>
        <h4>API Key Configuration</h4>
      </template>
      <el-form :model="apiForm" label-width="100px">
        <el-form-item label="API Keys">
          <el-input
            v-model="apiForm.apiKeys"
            type="textarea"
            :rows="3"
            placeholder="Please enter API keys, one per line. Multiple keys support parallel data fetching acceleration."
          />
          <div class="api-help-text">
            <el-text size="small" type="info">
              • One API key per line<br>
              • Only one API key per person can be used<br>
              • Multiple keys support parallel request acceleration<br>
              • Single key limit: 50 requests/minute
            </el-text>
          </div>
        </el-form-item>
      </el-form>
    </el-card>

    <!-- Faction Input -->
    <el-form :model="form" label-width="120px">
      <el-row :gutter="20">
        <el-col :span="12">
          <el-form-item label="Faction 1 ID">
            <el-input 
              v-model="form.faction1Id" 
              placeholder="Enter Faction 1 ID" 
            />
          </el-form-item>
        </el-col>
        <el-col :span="12">
          <el-form-item label="Faction 2 ID">
            <el-input 
              v-model="form.faction2Id" 
              placeholder="Enter Faction 2 ID" 
            />
          </el-form-item>
        </el-col>
      </el-row>
      <el-form-item>
        <el-button 
          type="primary" 
          @click="startComparison" 
          :loading="loading"
          :disabled="!canStartComparison"
        >
          Start Analysis
        </el-button>
        <el-button 
          v-if="loading"
          @click="stopComparison"
        >
          Stop Analysis
        </el-button>
      </el-form-item>
    </el-form>

    <!-- Progress Display -->
    <div v-if="loading || statusMessage" class="progress-section">
      <el-card class="progress-card">
        <template #header>
          <h4>Data Fetching Progress</h4>
        </template>
        
        <!-- Overall Progress -->
        <div class="progress-item">
          <div class="progress-label">Overall Progress</div>
          <el-progress 
            :percentage="overallProgress" 
            :status="loading ? '' : 'success'"
            :stroke-width="20"
          />
          <div class="progress-text">{{ progressText }}</div>
        </div>

        <!-- Detailed Progress -->
        <div v-if="detailedProgress.length > 0" class="detailed-progress">
          <el-collapse>
            <el-collapse-item title="Detailed Progress" name="details">
              <div v-for="item in detailedProgress" :key="item.key" class="progress-detail-item">
                <div class="progress-detail-label">{{ item.label }}</div>
                <el-progress 
                  :percentage="item.percentage" 
                  :status="item.status === 'active' ? '' : item.status"
                  :stroke-width="12"
                />
                <div class="progress-detail-text">{{ item.text }}</div>
              </div>
            </el-collapse-item>
          </el-collapse>
        </div>

        <!-- Status Message -->
        <el-alert 
          v-if="statusMessage"
          :title="statusMessage"
          :type="statusType"
          show-icon
          :closable="false"
          style="margin-top: 15px"
        />
      </el-card>
    </div>

    <!-- Cache Information -->
    <el-card v-if="cacheInfo.length > 0" class="cache-info-card" style="margin-top: 20px;">
      <template #header>
        <div class="cache-header">
          <h4>Cache Data Information ({{ cacheInfo.length }} items)</h4>
          <div>
            <el-button size="small" @click="clearCache">Clear Cache</el-button>
          </div>
        </div>
      </template>
      <el-collapse>
        <el-collapse-item title="View Cache Details" name="cache">
          <el-table :data="cacheInfo" size="small">
            <el-table-column prop="type" label="Data Type" width="150" />
            <el-table-column prop="id" label="ID" width="100" />
            <el-table-column prop="name" label="Name" />
            <el-table-column prop="lastUpdated" label="Last Updated" width="180" />
            <el-table-column prop="dataSize" label="Data Size" width="120" />
            <el-table-column label="Actions" width="150">
              <template #default="{ row }">
                <el-button size="small" @click="viewCacheData(row.key)">View</el-button>
                <el-button size="small" type="danger" @click="removeCacheItem(row.key)">Delete</el-button>
              </template>
            </el-table-column>
          </el-table>
        </el-collapse-item>
      </el-collapse>
    </el-card>

    <!-- Results Display Area -->
    <div v-if="comparisonResult" class="comparison-result">
      <el-card class="result-card" style="margin-top: 20px;">
        <template #header>
          <h3>Comparison Analysis Results</h3>
        </template>
        <div class="result-summary">
          <p>Data retrieval completed! Retrieved:</p>
          <ul>
            <li>Faction 1 ({{ comparisonResult.faction1.name }}): {{ getMemberCount(comparisonResult.faction1.members) }} members</li>
            <li>Faction 2 ({{ comparisonResult.faction2.name }}): {{ getMemberCount(comparisonResult.faction2.members) }} members</li>
            <li>RW Data: {{ comparisonResult.rwDataCount }} records ({{ comparisonResult.detailedRwCount }} detailed reports)</li>
            <li>Chain Data: {{ comparisonResult.chainDataCount }} records ({{ comparisonResult.detailedChainCount }} detailed reports)</li>
            <li>Personal Stats Data: {{ comparisonResult.personalStatsCount }} records</li>
          </ul>
          
          <div style="margin-top: 15px;">
            <h4>Data Details</h4>
            <el-row :gutter="20">
              <el-col :span="12">
                <h5>{{ comparisonResult.faction1.name }}</h5>
                <ul>
                  <li>RW: {{ comparisonResult.faction1.rankedWars.length }} records</li>
                  <li>Chain: {{ comparisonResult.faction1.chains.length }} records</li>
                  <li>Member Personal Data: {{ Object.keys(comparisonResult.faction1.personalStats).filter(id => comparisonResult.faction1.personalStats[id]).length }} records</li>
                </ul>
              </el-col>
              <el-col :span="12">
                <h5>{{ comparisonResult.faction2.name }}</h5>
                <ul>
                  <li>RW: {{ comparisonResult.faction2.rankedWars.length }} records</li>
                  <li>Chain: {{ comparisonResult.faction2.chains.length }} records</li>
                  <li>Member Personal Data: {{ Object.keys(comparisonResult.faction2.personalStats).filter(id => comparisonResult.faction2.personalStats[id]).length }} records</li>
                </ul>
              </el-col>
            </el-row>
          </div>
        </div>
      </el-card>


      <!-- PVP Win Rate Prediction -->
      <el-card v-if="comparisonResult.winRatePrediction" class="win-rate-card" style="margin-top: 20px;">
        <template #header>
          <h3>🏆 PVP Win Rate Prediction</h3>
        </template>
        <div class="win-rate-display">
          <el-row :gutter="20">
            <el-col :span="12">
              <div class="faction-win-rate faction-1">
                <h4>{{ comparisonResult.faction1.name }}</h4>
                <div class="win-percentage">
                  {{ comparisonResult.winRatePrediction.faction1WinRate }}%
                </div>
                <el-progress 
                  :percentage="comparisonResult.winRatePrediction.faction1WinRate" 
                  :stroke-width="20"
                  :color="comparisonResult.winRatePrediction.faction1WinRate > 50 ? '#67c23a' : '#f56c6c'"
                />
              </div>
            </el-col>
            <el-col :span="12">
              <div class="faction-win-rate faction-2">
                <h4>{{ comparisonResult.faction2.name }}</h4>
                <div class="win-percentage">
                  {{ comparisonResult.winRatePrediction.faction2WinRate }}%
                </div>
                <el-progress 
                  :percentage="comparisonResult.winRatePrediction.faction2WinRate" 
                  :stroke-width="20"
                  :color="comparisonResult.winRatePrediction.faction2WinRate > 50 ? '#67c23a' : '#f56c6c'"
                />
              </div>
            </el-col>
          </el-row>
          
          <!-- 24-Hour Battle Win Rate Table -->
          <div v-if="comparisonResult.winRatePrediction.hourlyWinRates" class="hourly-win-rates" style="margin-top: 30px;">
            <el-collapse>
              <el-collapse-item name="hourly-rates">
                <template #title>
                  <h4>⏰ 24-Hour Battle Win Rate Details (Average Win Rate: {{ comparisonResult.winRatePrediction.faction1WinRate }}% : {{ comparisonResult.winRatePrediction.faction2WinRate }}%)</h4>
                </template>
                
                <div class="hourly-explanation" style="margin-bottom: 15px;">
                  <el-alert 
                    title="Explanation" 
                    type="info" 
                    :closable="false"
                    description="Win rates calculated based on member sleep schedules for different battle start times. 100% combat power during active hours, 30% during sleep hours. Members without data estimated using faction average sleep times."
                  />
                </div>
                
                <!-- Win Rate Trend Chart Area -->
                <div class="hourly-chart" style="margin-bottom: 20px;">
                  <h5>Win Rate Trend Chart</h5>
                  <div class="chart-container" style="height: 200px; position: relative; border: 1px solid #e4e7ed; border-radius: 4px; padding: 10px;">
                    <div class="chart-axis" style="position: absolute; bottom: 0; left: 0; right: 0; height: 1px; background: #ddd;"></div>
                    <div class="chart-bars" style="height: 180px; display: flex; align-items: end; justify-content: space-between;">
                      <div 
                        v-for="hour in comparisonResult.winRatePrediction.hourlyWinRates" 
                        :key="hour.hour"
                        class="chart-bar" 
                        :style="{
                          height: hour.faction1WinRate + '%',
                          width: '3.8%',
                          backgroundColor: hour.faction1WinRate > 50 ? '#67c23a' : '#f56c6c',
                          opacity: 0.8,
                          borderRadius: '2px 2px 0 0',
                          position: 'relative'
                        }"
                        :title="`${hour.timeDisplay}: ${hour.faction1WinRate}%`"
                      >
                        <div style="position: absolute; bottom: -20px; font-size: 10px; text-align: center; width: 100%; color: #666;">
                          {{ hour.hour % 4 === 0 ? hour.hour : '' }}
                        </div>
                      </div>
                    </div>
                  </div>
                </div>
                
                <!-- Detailed Data Table -->
                <el-table 
                  :data="comparisonResult.winRatePrediction.hourlyWinRates" 
                  size="small" 
                  max-height="400"
                  stripe
                >
                  <el-table-column prop="timeDisplay" label="Battle Time" width="100" align="center" />
                  <el-table-column :label="`${comparisonResult.faction1.name} Win Rate`" width="120" align="center">
                    <template #default="{ row }">
                      <el-tag 
                        :type="row.faction1WinRate > 70 ? 'success' : row.faction1WinRate > 40 ? 'warning' : 'danger'" 
                        size="small"
                      >
                        {{ row.faction1WinRate }}%
                      </el-tag>
                    </template>
                  </el-table-column>
                  <el-table-column :label="`${comparisonResult.faction2.name} Win Rate`" width="120" align="center">
                    <template #default="{ row }">
                      <el-tag 
                        :type="row.faction2WinRate > 70 ? 'success' : row.faction2WinRate > 40 ? 'warning' : 'danger'" 
                        size="small"
                      >
                        {{ row.faction2WinRate }}%
                      </el-tag>
                    </template>
                  </el-table-column>
                  <el-table-column :label="`${comparisonResult.faction1.name} Active Members`" width="120" align="center">
                    <template #default="{ row }">
                      {{ row.faction1Strength.activeMemberCount }} / {{ row.faction1Strength.memberCount }}
                    </template>
                  </el-table-column>
                  <el-table-column :label="`${comparisonResult.faction2.name} Active Members`" width="120" align="center">
                    <template #default="{ row }">
                      {{ row.faction2Strength.activeMemberCount }} / {{ row.faction2Strength.memberCount }}
                    </template>
                  </el-table-column>
                  <el-table-column :label="`${comparisonResult.faction1.name} Strength`" width="120" align="center">
                    <template #default="{ row }">
                      {{ Math.round(row.faction1Strength.effectiveCombatPower) }}
                    </template>
                  </el-table-column>
                  <el-table-column :label="`${comparisonResult.faction2.name} Strength`" width="120" align="center">
                    <template #default="{ row }">
                      {{ Math.round(row.faction2Strength.effectiveCombatPower) }}
                    </template>
                  </el-table-column>
                  <el-table-column label="Strength Ratio" width="100" align="center">
                    <template #default="{ row }">
                      <span :style="{ color: row.faction1Score > row.faction2Score ? '#67c23a' : '#f56c6c' }">
                        {{ row.faction2Score > 0 ? (row.faction1Score / row.faction2Score).toFixed(2) : '∞' }}:1
                      </span>
                    </template>
                  </el-table-column>
                </el-table>
                
                <!-- Optimal Battle Time Recommendations -->
                <div class="best-time-suggestion" style="margin-top: 20px;">
                  <el-card>
                    <template #header>
                      <h5>🎯 Optimal Battle Time Recommendations</h5>
                    </template>
                    <el-row :gutter="20">
                      <el-col :span="12">
                        <div class="faction-best-times">
                          <h6>{{ comparisonResult.faction1.name }} Optimal Time Slots:</h6>
                          <div class="best-times">
                            <el-tag 
                              v-for="hour in getBestTimesForFaction(comparisonResult.winRatePrediction.hourlyWinRates, 1)" 
                              :key="hour.hour"
                              type="success" 
                              size="small" 
                              style="margin: 2px;"
                            >
                              {{ hour.timeDisplay }} ({{ hour.faction1WinRate }}%)
                            </el-tag>
                          </div>
                        </div>
                      </el-col>
                      <el-col :span="12">
                        <div class="faction-best-times">
                          <h6>{{ comparisonResult.faction2.name }} Optimal Time Slots:</h6>
                          <div class="best-times">
                            <el-tag 
                              v-for="hour in getBestTimesForFaction(comparisonResult.winRatePrediction.hourlyWinRates, 2)" 
                              :key="hour.hour"
                              type="success" 
                              size="small" 
                              style="margin: 2px;"
                            >
                              {{ hour.timeDisplay }} ({{ hour.faction2WinRate }}%)
                            </el-tag>
                          </div>
                        </div>
                      </el-col>
                    </el-row>
                  </el-card>
                </div>
              </el-collapse-item>
            </el-collapse>
          </div>
          
          <div class="analysis-text">
            <el-card>
              <template #header>
                <h4>Detailed Analysis</h4>
              </template>
              <div class="analysis-details">
                <el-row :gutter="20">
                  <el-col :span="12">
                    <div class="faction-analysis">
                      <h5>{{ comparisonResult.winRatePrediction.analysisData.faction1.name }}</h5>
                      <ul>
                        <li>Overall Strength Score: <strong>{{ comparisonResult.winRatePrediction.analysisData.faction1.combatPowerScore }}</strong></li>
                        <li>Average BS: <strong>{{ comparisonResult.winRatePrediction.analysisData.faction1.averageBS }}</strong></li>
                        <li>Activity Score: <strong>{{ comparisonResult.winRatePrediction.analysisData.faction1.activityScore }}</strong></li>
                        <li>Member Count: <strong>{{ comparisonResult.winRatePrediction.analysisData.faction1.memberCount }}</strong> members</li>
                        <li>Overall Rating: <strong>{{ comparisonResult.winRatePrediction.analysisData.faction1.score }}</strong></li>
                      </ul>
                    </div>
                  </el-col>
                  <el-col :span="12">
                    <div class="faction-analysis">
                      <h5>{{ comparisonResult.winRatePrediction.analysisData.faction2.name }}</h5>
                      <ul>
                        <li>Overall Strength Score: <strong>{{ comparisonResult.winRatePrediction.analysisData.faction2.combatPowerScore }}</strong></li>
                        <li>Average BS: <strong>{{ comparisonResult.winRatePrediction.analysisData.faction2.averageBS }}</strong></li>
                        <li>Activity Score: <strong>{{ comparisonResult.winRatePrediction.analysisData.faction2.activityScore }}</strong></li>
                        <li>Member Count: <strong>{{ comparisonResult.winRatePrediction.analysisData.faction2.memberCount }}</strong> members</li>
                        <li>Overall Rating: <strong>{{ comparisonResult.winRatePrediction.analysisData.faction2.score }}</strong></li>
                      </ul>
                    </div>
                  </el-col>
                </el-row>
              </div>
            </el-card>
          </div>
        </div>
      </el-card>

      <!-- Faction Strength Detailed Analysis -->
      <el-card v-if="comparisonResult.faction1Analysis && comparisonResult.faction2Analysis" class="strength-analysis-card" style="margin-top: 20px;">
        <template #header>
          <h3>📊 Faction Strength Detailed Analysis</h3>
        </template>
        
        <!-- Overall Comparison -->
        <div class="overall-comparison">
          <h4>Overall Strength Comparison</h4>
          <el-table :data="[
            {
              metric: 'Average BS',
              faction1: formatBSValue(Math.round(comparisonResult.faction1Analysis.averageBS)),
              faction2: formatBSValue(Math.round(comparisonResult.faction2Analysis.averageBS)),
              faction1Raw: Math.round(comparisonResult.faction1Analysis.averageBS),
              faction2Raw: Math.round(comparisonResult.faction2Analysis.averageBS)
            },
            {
              metric: 'Overall Strength Score',
              faction1: Math.round(comparisonResult.faction1Analysis.averageCombatPower),
              faction2: Math.round(comparisonResult.faction2Analysis.averageCombatPower)
            },
            {
              metric: 'Four-Month Average Attacks',
              faction1: Math.round(comparisonResult.faction1Analysis.averageAttacksFourMonth),
              faction2: Math.round(comparisonResult.faction2Analysis.averageAttacksFourMonth)
            },
            {
              metric: 'Recent Month Average Attacks',
              faction1: Math.round(comparisonResult.faction1Analysis.averageAttacksPerMonth),
              faction2: Math.round(comparisonResult.faction2Analysis.averageAttacksPerMonth)
            },
            {
              metric: 'Overall Activity Score',
              faction1: Math.round(comparisonResult.faction1Analysis.averageActivityScore),
              faction2: Math.round(comparisonResult.faction2Analysis.averageActivityScore)
            },
            {
              metric: 'Member Count',
              faction1: comparisonResult.faction1Analysis.memberCount,
              faction2: comparisonResult.faction2Analysis.memberCount
            }
          ]" style="width: 100%">
            <el-table-column prop="metric" label="Metric" width="200" />
            <el-table-column :label="comparisonResult.faction1.name" align="center">
              <template #default="{ row }">
                <span :style="{ color: (row.faction1Raw || row.faction1) > (row.faction2Raw || row.faction2) ? '#67c23a' : '#909399' }">
                  {{ row.faction1 }}
                </span>
              </template>
            </el-table-column>
            <el-table-column :label="comparisonResult.faction2.name" align="center">
              <template #default="{ row }">
                <span :style="{ color: (row.faction2Raw || row.faction2) > (row.faction1Raw || row.faction1) ? '#67c23a' : '#909399' }">
                  {{ row.faction2 }}
                </span>
              </template>
            </el-table-column>
          </el-table>
        </div>

        <!-- Member Detailed Analysis -->
        <div class="member-analysis" style="margin-top: 30px;">
          <h4>Member Detailed Analysis</h4>
          <el-tabs type="border-card">
            <el-tab-pane :label="comparisonResult.faction1.name">
              <div class="member-count-info">
                Total {{ comparisonResult.faction1Analysis.memberAnalysis.length }} members
              </div>
              <el-table 
                :data="comparisonResult.faction1Analysis.memberAnalysis" 
                size="small"
                max-height="600"
                :default-sort="{ prop: 'combatPowerScore', order: 'descending' }"
              >
                <el-table-column prop="name" label="Member Name" width="120" fixed="left" />
                <el-table-column prop="id" label="ID" width="80" />
                <el-table-column prop="combatPowerScore" label="Overall Strength Score" width="100" align="center" sortable>
                  <template #default="{ row }">
                    <el-tag :type="row.combatPowerScore > 300 ? 'danger' : row.combatPowerScore > 200 ? 'warning' : row.combatPowerScore > 100 ? 'success' : 'info'" size="small">
                      {{ row.combatPowerScore }}
                    </el-tag>
                  </template>
                </el-table-column>
                <el-table-column prop="estimatedBS" label="Estimated BS" width="100" align="center" sortable>
                  <template #default="{ row }">
                    <el-tag :type="row.confidence === 'high' ? 'success' : row.confidence === 'medium' ? 'warning' : 'info'" size="small">
                      {{ formatBSValue(row.estimatedBS) }}
                    </el-tag>
                  </template>
                </el-table-column>
                <el-table-column prop="fourMonthAttacks" label="4-Month Attacks" width="100" align="center" sortable />
                <el-table-column prop="oneMonthAttacks" label="1-Month Attacks" width="100" align="center" sortable />
                <el-table-column prop="hosPercentage" label="HOS Percentage" width="90" align="center" sortable>
                  <template #default="{ row }">
                    <span :style="{ color: row.hosPercentage > 20 ? '#67c23a' : '#909399' }">
                      {{ row.hosPercentage.toFixed(1) }}%
                    </span>
                  </template>
                </el-table-column>
                <el-table-column prop="revengePercentage" label="Retaliation Percentage" width="90" align="center" sortable>
                  <template #default="{ row }">
                    <span :style="{ color: row.revengePercentage > 10 ? '#f56c6c' : '#909399' }">
                      {{ row.revengePercentage.toFixed(1) }}%
                    </span>
                  </template>
                </el-table-column>
                <el-table-column label="Sleep Period" min-width="120">
                  <template #default="{ row }">
                    <span v-if="row.sleepPeriod" class="sleep-period">
                      {{ formatSleepPeriod(row.sleepPeriod) }}
                    </span>
                    <span v-else style="color: #909399;">No Data</span>
                  </template>
                </el-table-column>
                <el-table-column prop="activityScore" label="Activity Score" width="100" align="center" sortable>
                  <template #default="{ row }">
                    <el-tag :type="row.activityScore > 100 ? 'success' : row.activityScore > 50 ? 'warning' : 'info'" size="small">
                      {{ Math.round(row.activityScore) }}
                    </el-tag>
                  </template>
                </el-table-column>
              </el-table>
            </el-tab-pane>
            <el-tab-pane :label="comparisonResult.faction2.name">
              <div class="member-count-info">
                Total {{ comparisonResult.faction2Analysis.memberAnalysis.length }} members
              </div>
              <el-table 
                :data="comparisonResult.faction2Analysis.memberAnalysis" 
                size="small"
                max-height="600"
                :default-sort="{ prop: 'combatPowerScore', order: 'descending' }"
              >
                <el-table-column prop="name" label="Member Name" width="120" fixed="left" />
                <el-table-column prop="id" label="ID" width="80" />
                <el-table-column prop="combatPowerScore" label="Overall Strength Score" width="100" align="center" sortable>
                  <template #default="{ row }">
                    <el-tag :type="row.combatPowerScore > 300 ? 'danger' : row.combatPowerScore > 200 ? 'warning' : row.combatPowerScore > 100 ? 'success' : 'info'" size="small">
                      {{ row.combatPowerScore }}
                    </el-tag>
                  </template>
                </el-table-column>
                <el-table-column prop="estimatedBS" label="Estimated BS" width="100" align="center" sortable>
                  <template #default="{ row }">
                    <el-tag :type="row.confidence === 'high' ? 'success' : row.confidence === 'medium' ? 'warning' : 'info'" size="small">
                      {{ formatBSValue(row.estimatedBS) }}
                    </el-tag>
                  </template>
                </el-table-column>
                <el-table-column prop="fourMonthAttacks" label="4-Month Attacks" width="100" align="center" sortable />
                <el-table-column prop="oneMonthAttacks" label="1-Month Attacks" width="100" align="center" sortable />
                <el-table-column prop="hosPercentage" label="HOS Percentage" width="80" align="center" sortable>
                  <template #default="{ row }">
                    <span :style="{ color: row.hosPercentage > 20 ? '#67c23a' : '#909399' }">
                      {{ row.hosPercentage.toFixed(1) }}%
                    </span>
                  </template>
                </el-table-column>
                <el-table-column prop="revengePercentage" label="Revenge Percentage" width="80" align="center" sortable>
                  <template #default="{ row }">
                    <span :style="{ color: row.revengePercentage > 10 ? '#f56c6c' : '#909399' }">
                      {{ row.revengePercentage.toFixed(1) }}%
                    </span>
                  </template>
                </el-table-column>
                <el-table-column label="Sleep Period" min-width="120">
                  <template #default="{ row }">
                    <span v-if="row.sleepPeriod" class="sleep-period">
                      {{ formatSleepPeriod(row.sleepPeriod) }}
                    </span>
                    <span v-else style="color: #909399;">No Data</span>
                  </template>
                </el-table-column>
                <el-table-column prop="activityScore" label="Activity Score" width="100" align="center" sortable>
                  <template #default="{ row }">
                    <el-tag :type="row.activityScore > 100 ? 'success' : row.activityScore > 50 ? 'warning' : 'info'" size="small">
                      {{ Math.round(row.activityScore) }}
                    </el-tag>
                  </template>
                </el-table-column>
              </el-table>
            </el-tab-pane>
          </el-tabs>
        </div>
      </el-card>
    </div>
  </el-card>
  
  <!-- Cache Data View Dialog -->
  <el-dialog
    v-model="showCacheDataDialog"
    title="Cache Data Details"
    width="80%"
    :show-close="true"
  >
    <div class="cache-data-content">
      <el-tabs>
        <el-tab-pane label="Formatted JSON" name="formatted">
          <pre class="json-content">{{ formatJsonData(selectedCacheData) }}</pre>
        </el-tab-pane>
        <el-tab-pane label="Raw Data" name="raw">
          <el-input
            v-model="rawJsonData"
            type="textarea"
            :rows="20"
            readonly
            style="font-family: 'Courier New', monospace;"
          />
        </el-tab-pane>
      </el-tabs>
    </div>
    <template #footer>
      <el-button @click="showCacheDataDialog = false">Close</el-button>
      <el-button type="primary" @click="copyCacheData">Copy Data</el-button>
    </template>
  </el-dialog>
</template>

<script setup>
import { ref, reactive, computed, onMounted } from 'vue'
import axios from 'axios'
import { ElMessage } from 'element-plus'

const API_BASE_URL = 'https://api.torn.com/v2'
const RATE_LIMIT_PER_MINUTE = 50
const RATE_LIMIT_INTERVAL = 60000 // 1 minute

// BS prediction algorithm constants
const BS_CONSTANTS = {
  L: [2, 2.8, 3.2, 3.2, 3.6, 3.8, 3.7, 4, 4.8, 4.8, 5.2, 5.2, 5.4, 5.8, 5.8, 6, 6.4, 6.6, 6.8, 7, 7, 7, 7, 7.3, 8],
  W: [200, 500, 1000, 2000, 2750, 3000, 3500, 4000, 6000, 7000, 8000, 11000, 12420, 18000, 18100, 24140, 31260, 36610, 46640, 56520, 67775, 84535, 106305, 100000, Infinity],
  E: [5, 5, 5, 5, 5, 5, 10, 10, 10, 10, 10, 10, 25, 25, 25, 25, 25, 25, 25, 50, 50, 50, 50, 50, 50], // energy consumption per gym
  // new training formula attribute-specific constants
  STAT_CONSTANTS: {
    strength: { A: 1600, B: 1700, C: 700 },
    speed: { A: 1600, B: 2000, C: 1350 },
    dexterity: { A: 1800, B: 1500, C: 1000 },
    defense: { A: 2100, B: -600, C: 1500 }
  }
}

// form data
const apiForm = reactive({
  apiKeys: ''
})

const form = reactive({
  faction1Id: '',
  faction2Id: ''
})

// state management
const loading = ref(false)
const statusMessage = ref('')
const statusType = ref('info')
const progressText = ref('')
const overallProgress = ref(0)
const detailedProgress = ref([])
const comparisonResult = ref(null)
const abortController = ref(null)

// cache management
const dataCache = ref(new Map())
const cacheInfo = ref([])

// cache data view
const showCacheDataDialog = ref(false)
const selectedCacheData = ref(null)
const rawJsonData = ref('')

// API key management
const apiKeyPool = ref([])
const apiKeyUsage = ref(new Map()) // track usage of each API key

// computed properties
const canStartComparison = computed(() => {
  return form.faction1Id && form.faction2Id && getValidApiKeys().length > 0
})

// get valid API key list
const getValidApiKeys = () => {
  return apiForm.apiKeys
    .split('\n')
    .map(key => key.trim())
    .filter(key => key.length > 0)
}

// API request queue management
class ApiRequestQueue {
  constructor(apiKeys) {
    this.apiKeys = apiKeys
    this.keyUsage = new Map()
    this.requestQueue = []
    this.isProcessing = false
    
    // initialize usage statistics for each API key
    apiKeys.forEach(key => {
      this.keyUsage.set(key, {
        requests: [],
        lastReset: Date.now()
      })
    })
  }

  // get available API key
  getAvailableApiKey() {
    const now = Date.now()
    
    for (const apiKey of this.apiKeys) {
      const usage = this.keyUsage.get(apiKey)
      
      // clear requests older than 1 minute
      usage.requests = usage.requests.filter(time => now - time < RATE_LIMIT_INTERVAL)
      
      // if the key has not reached its limit
      if (usage.requests.length < RATE_LIMIT_PER_MINUTE) {
        return apiKey
      }
    }
    
    return null
  }

  // record API key usage
  recordApiKeyUsage(apiKey) {
    const usage = this.keyUsage.get(apiKey)
    if (usage) {
      usage.requests.push(Date.now())
    }
  }

  // add request to queue
  async addRequest(requestFn) {
    return new Promise((resolve, reject) => {
      this.requestQueue.push({ requestFn, resolve, reject })
      this.processQueue()
    })
  }

  // process request queue
  async processQueue() {
    if (this.isProcessing || this.requestQueue.length === 0) {
      return
    }

    this.isProcessing = true

    while (this.requestQueue.length > 0) {
      const availableKey = this.getAvailableApiKey()
      
      if (!availableKey) {
        // 等待一段时间再重试
        await new Promise(resolve => setTimeout(resolve, 1000))
        continue
      }

      const { requestFn, resolve, reject } = this.requestQueue.shift()
      
      try {
        this.recordApiKeyUsage(availableKey)
        const result = await requestFn(availableKey)
        resolve(result)
      } catch (error) {
        reject(error)
      }

      // 添加小延迟避免过快请求
      await new Promise(resolve => setTimeout(resolve, 100))
    }

    this.isProcessing = false
  }
}

// 重试配置
const RETRY_CONFIG = {
  maxRetries: 3, // 最大重试次数
  baseDelay: 1000, // 基础延迟（毫秒）
  maxDelay: 30000, // 最大延迟（毫秒）
  exponentialBase: 2, // 指数退避基数
  jitterFactor: 0.1 // 抖动因子，避免所有请求同时重试
}

// 判断错误是否应该重试
const shouldRetryError = (error) => {
  // 如果是取消请求，不重试
  if (error.name === 'AbortError' || error.message === 'Request canceled') {
    return false
  }
  
  // 如果是网络错误，重试
  if (!error.response && (error.code === 'ECONNRESET' || error.code === 'ENOTFOUND' || error.code === 'ECONNREFUSED')) {
    return true
  }
  
  // 如果有响应，根据状态码判断
  if (error.response) {
    const status = error.response.status
    // 5xx 服务器错误 - 重试
    if (status >= 500) return true
    // 429 请求过多 - 重试
    if (status === 429) return true
    // 408 请求超时 - 重试
    if (status === 408) return true
    // 502, 503, 504 网关错误 - 重试
    if ([502, 503, 504].includes(status)) return true
    
    // 检查Torn API特定错误
    if (error.response.data && error.response.data.error) {
      const tornError = error.response.data.error
      // API密钥过期或无效 - 不重试
      if ([1, 2].includes(tornError.code)) return false
      // 权限不足 - 不重试
      if ([7, 8].includes(tornError.code)) return false
      // 用户不存在或帮派不存在 - 不重试
      if ([6, 23].includes(tornError.code)) return false
      // 其他错误可以重试
      return true
    }
    
    // 4xx 客户端错误一般不重试（除了上面特殊的几种）
    if (status >= 400 && status < 500) return false
  }
  
  return true // 其他未知错误，默认重试
}

// 计算重试延迟（指数退避 + 抖动）
const calculateRetryDelay = (attempt) => {
  const baseDelay = RETRY_CONFIG.baseDelay * Math.pow(RETRY_CONFIG.exponentialBase, attempt)
  const jitter = baseDelay * RETRY_CONFIG.jitterFactor * (Math.random() * 2 - 1)
  const delay = Math.min(baseDelay + jitter, RETRY_CONFIG.maxDelay)
  return Math.max(delay, 0)
}

// 带重试的API请求函数
const fetchApiWithRetry = async (endpoint, apiKey, options = {}) => {
  const { maxRetries = RETRY_CONFIG.maxRetries, context = 'API请求' } = options
  let lastError = null
  
  for (let attempt = 0; attempt <= maxRetries; attempt++) {
    try {
      // 检查是否被取消
      if (abortController.value?.signal.aborted) {
        throw new Error('请求被取消')
      }
      
      const result = await fetchApi(endpoint, apiKey)
      
      // 如果成功且之前有重试，记录成功信息
      if (attempt > 0) {
        console.log(`${context} 重试成功: ${endpoint} (第${attempt}次重试)`)
      }
      
      return result
    } catch (error) {
      lastError = error
      
      // 如果是最后一次尝试，或者错误不应该重试，直接抛出
      if (attempt === maxRetries || !shouldRetryError(error)) {
        if (attempt > 0) {
          console.error(`${context} 重试失败，已达到最大重试次数: ${endpoint}`, error)
        }
        throw error
      }
      
      // 计算重试延迟
      const delay = calculateRetryDelay(attempt)
      console.warn(`${context} 失败，将在 ${Math.round(delay/1000)}s 后进行第${attempt + 1}次重试: ${endpoint}`, error.message)
      
      // 更新状态消息显示重试信息
      if (statusMessage.value && !statusMessage.value.includes('Canceled')) {
        statusMessage.value = `${context} failed, retrying... (attempt ${attempt + 1})`
      }
      
      // 等待重试延迟
      await new Promise(resolve => setTimeout(resolve, delay))
      
      // 再次检查是否被取消
      if (abortController.value?.signal.aborted) {
        throw new Error('请求被取消')
      }
    }
  }
  
  throw lastError
}

// API请求函数（保持原有逻辑不变）
const fetchApi = async (endpoint, apiKey) => {
  let url = `${API_BASE_URL}${endpoint}`
  if (url.includes('?')) {
    url += `&key=${apiKey}`
  } else {
    url += `?key=${apiKey}`
  }
  
  console.log(`发起API请求: ${url.replace(/key=[^&]+/, 'key=***')}`)
  
  try {
    const response = await axios.get(url, {
      signal: abortController.value?.signal,
      timeout: 30000 // 30秒超时
    })
    
    console.log(`API响应成功:`, response.data)
    
    if (response.data.error) {
      throw new Error(`Torn API 错误: ${response.data.error.error} (代码: ${response.data.error.code})`)
    }
    
    return response.data
  } catch (error) {
    if (error.name === 'AbortError') {
      throw new Error('请求被取消')
    }
    
    console.error(`API请求失败:`, error)
    
    if (error.response) {
      const errorData = error.response.data
      console.error(`API错误响应:`, errorData)
      let errorMessage = `API请求失败，状态码: ${error.response.status}.`
      if (errorData && errorData.error && errorData.error.error) {
        errorMessage += ` 错误: ${errorData.error.error}`
      }
      throw new Error(errorMessage)
    }
    throw error
  }
}

// 缓存相关函数
const getCacheKey = (type, id, timeRange = '') => {
  return `${type}_${id}_${timeRange}`
}

const getCachedData = (key) => {
  const cached = dataCache.value.get(key)
  if (cached && Date.now() - cached.timestamp < 30 * 60 * 1000) { // 30分钟缓存
    return cached.data
  }
  return null
}

const setCachedData = (key, data, name = '') => {
  dataCache.value.set(key, {
    data,
    timestamp: Date.now(),
    name
  })
  updateCacheInfo()
}

const updateCacheInfo = () => {
  cacheInfo.value = Array.from(dataCache.value.entries()).map(([key, value]) => {
    const [type, id] = key.split('_')
    return {
      key,
      type: type === 'faction' ? '帮派信息' : 
            type === 'members' ? '成员列表' :
            type === 'personalstats' ? '个人数据' :
            type === 'rankedwars' ? 'RW数据' :
            type === 'chains' ? 'Chain数据' : type,
      id,
      name: value.name || '未知',
      lastUpdated: new Date(value.timestamp).toLocaleString('zh-CN'),
      dataSize: getDataSize(value.data)
    }
  })
}

const getDataSize = (data) => {
  const size = JSON.stringify(data).length
  if (size < 1024) return `${size} B`
  if (size < 1024 * 1024) return `${(size / 1024).toFixed(1)} KB`
  return `${(size / 1024 / 1024).toFixed(1)} MB`
}

const clearCache = () => {
  dataCache.value.clear()
  updateCacheInfo()
  ElMessage.success('缓存已清空')
}

const removeCacheItem = (key) => {
  dataCache.value.delete(key)
  updateCacheInfo()
  ElMessage.success('缓存项已删除')
}

// 查看缓存数据
const viewCacheData = (key) => {
  const cached = dataCache.value.get(key)
  if (cached) {
    selectedCacheData.value = cached.data
    rawJsonData.value = JSON.stringify(cached.data, null, 2)
    showCacheDataDialog.value = true
  }
}

// 格式化JSON数据显示
const formatJsonData = (data) => {
  if (!data) return ''
  try {
    return JSON.stringify(data, null, 2)
  } catch (error) {
    return String(data)
  }
}

// 复制缓存数据
const copyCacheData = () => {
  if (rawJsonData.value) {
    navigator.clipboard.writeText(rawJsonData.value).then(() => {
      ElMessage.success('数据已复制到剪贴板')
    }).catch(() => {
      ElMessage.error('复制失败')
    })
  }
}

// 使用旧版训练公式计算单次训练增益（2022年8月2日之前）
const calculateTrainingGainOld = (currentStats, happy, gymLevel, energyPerTrain, statType = 'strength') => {
  // 获取属性特定常数
  const statConstants = BS_CONSTANTS.STAT_CONSTANTS[statType]
  const A = statConstants.A
  const B = statConstants.B
  
  // S: 旧版硬上限50m
  const S = currentStats <= 50000000 ? currentStats : 50000000
  
  // H: 当前快乐值
  const H = Math.max(0, Math.min(happy, 99999))
  
  // G: 健身房点数
  const G = BS_CONSTANTS.L[gymLevel]
  
  // E: 每次训练消耗的能量
  const E = energyPerTrain
  
  // 旧版公式的各个部分
  const lnTerm = Math.log(1 + H/250)
  const roundedLn = Math.round(lnTerm * 10000) / 10000 // 四位小数
  const multiplier = Math.round((1 + 0.07 * roundedLn) * 10000) / 10000 // 四位小数
  const statComponent = S * multiplier
  
  const happyComponent = 8 * Math.pow(H, 1.05)
  const happyAdjustment = H < 99999 ? (1 - Math.pow(H/99999, 2)) * A : 0
  
  const baseGain = (statComponent + happyComponent + happyAdjustment + B) * (1/200000) * G * E
  
  return Math.max(0, baseGain)
}

// 使用新版训练公式计算单次训练增益（2022年8月2日之后）
const calculateTrainingGainNew = (currentStats, happy, gymLevel, energyPerTrain, statType = 'strength') => {
  // 获取属性特定常数
  const statConstants = BS_CONSTANTS.STAT_CONSTANTS[statType]
  const A = statConstants.A
  const B = statConstants.B
  
  // S: 新版公式，超过50m时有对数衰减
  let S
  if (currentStats <= 50000000) {
    S = currentStats
  } else {
    S = 50000000 + (currentStats - 50000000) / (8.77635 * Math.log(currentStats))
  }
  
  // H: 当前快乐值
  const H = Math.max(0, Math.min(happy, 99999))
  
  // G: 健身房点数
  const G = BS_CONSTANTS.L[gymLevel]
  
  // E: 每次训练消耗的能量
  const E = energyPerTrain
  
  // 新版公式的各个部分
  const lnTerm = Math.log(1 + H/250)
  const roundedLn = Math.round(lnTerm * 10000) / 10000 // 四位小数
  const multiplier = Math.round((1 + 0.07 * roundedLn) * 10000) / 10000 // 四位小数
  const statComponent = S * multiplier
  
  const happyComponent = 8 * Math.pow(H, 1.05)
  const happyAdjustment = H < 99999 ? (1 - Math.pow(H/99999, 2)) * A : 0
  
  const baseGain = (statComponent + happyComponent + happyAdjustment + B) * (1/200000) * G * E
  
  return Math.max(0, baseGain)
}

// 模拟健身房锻炼（根据账户年龄使用旧版/新版公式）
const simulateGymTraining = (totalEnergy, stats, profile) => {
  // 计算能量分配
  const now = Math.floor(Date.now() / 1000)
  const formulaChangeDate = Math.floor(new Date('2022-08-02').getTime() / 1000)
  const accountAge = profile.age || 100
  const accountCreationTimestamp = now - (accountAge * 86400)
  
  let oldEnergy = 0
  let newEnergy = totalEnergy
  
  if (accountCreationTimestamp < formulaChangeDate) {
    // 账户在公式更新前创建，需要分配能量
    const daysBeforeChange = (formulaChangeDate - accountCreationTimestamp) / 86400
    const daysAfterChange = accountAge - daysBeforeChange
    
    const oldEnergyRatio = daysBeforeChange / accountAge
    const newEnergyRatio = daysAfterChange > 0 ? daysAfterChange / accountAge : 0
    
    oldEnergy = Math.floor(totalEnergy * oldEnergyRatio)
    newEnergy = totalEnergy - oldEnergy
  }
  
  // 初始化变量 - 分别跟踪四个属性
  let strengthStats = 0
  let speedStats = 0
  let dexterityStats = 0
  let defenseStats = 0
  
  const currentHappy = 5000 // 恒定快乐值
  
  // 属性训练顺序
  const statTypes = ['strength', 'speed', 'dexterity', 'defense']
  let currentStatIndex = 0
  
  // 初始化健身房状态变量
  let currentGym = 0
  let gymCapacityLeft = BS_CONSTANTS.W[0]
  
  // 第一阶段：使用旧版公式
  if (oldEnergy > 0) {
    let remainingEnergy = oldEnergy
    
    while (remainingEnergy > 0 && currentGym < BS_CONSTANTS.L.length) {
      const energyPerTrain = BS_CONSTANTS.E[currentGym]
      
      const maxTrainsByEnergy = Math.floor(remainingEnergy / energyPerTrain)
      const maxTrainsByCapacity = Math.floor(gymCapacityLeft / energyPerTrain)
      const actualTrains = Math.min(maxTrainsByEnergy, maxTrainsByCapacity, 10000)
      
      if (actualTrains <= 0) break
      
      for (let train = 0; train < actualTrains; train++) {
        if (remainingEnergy < energyPerTrain) break
        
        const currentStat = statTypes[currentStatIndex]
        let currentStatValue
        
        if (currentStat === 'strength') currentStatValue = strengthStats
        else if (currentStat === 'speed') currentStatValue = speedStats
        else if (currentStat === 'dexterity') currentStatValue = dexterityStats
        else currentStatValue = defenseStats
        
        // 使用旧版公式
        const gain = calculateTrainingGainOld(
          currentStatValue,
          currentHappy,
          currentGym,
          energyPerTrain,
          currentStat
        )
        
        // 更新对应属性的值
        if (currentStat === 'strength') strengthStats += gain
        else if (currentStat === 'speed') speedStats += gain
        else if (currentStat === 'dexterity') dexterityStats += gain
        else defenseStats += gain
        
        remainingEnergy -= energyPerTrain
        gymCapacityLeft -= energyPerTrain
        currentStatIndex = (currentStatIndex + 1) % 4
      }
      
      if (gymCapacityLeft <= energyPerTrain && currentGym < BS_CONSTANTS.L.length - 1) {
        currentGym++
        gymCapacityLeft = BS_CONSTANTS.W[currentGym]
      } else if (actualTrains === 0) {
        break
      }
    }
  }
  
  // 第二阶段：使用新版公式
  if (newEnergy > 0) {
    let remainingEnergy = newEnergy
    
    // 如果第一阶段没有训练，确保健身房状态已正确初始化
    // （这些变量已经在上面初始化过了）
    
    while (remainingEnergy > 0 && currentGym < BS_CONSTANTS.L.length) {
      const energyPerTrain = BS_CONSTANTS.E[currentGym]
      
      const maxTrainsByEnergy = Math.floor(remainingEnergy / energyPerTrain)
      const maxTrainsByCapacity = Math.floor(gymCapacityLeft / energyPerTrain)
      const actualTrains = Math.min(maxTrainsByEnergy, maxTrainsByCapacity, 10000)
      
      if (actualTrains <= 0) break
      
      for (let train = 0; train < actualTrains; train++) {
        if (remainingEnergy < energyPerTrain) break
        
        const currentStat = statTypes[currentStatIndex]
        let currentStatValue
        
        if (currentStat === 'strength') currentStatValue = strengthStats
        else if (currentStat === 'speed') currentStatValue = speedStats
        else if (currentStat === 'dexterity') currentStatValue = dexterityStats
        else currentStatValue = defenseStats
        
        // 使用新版公式
        const gain = calculateTrainingGainNew(
          currentStatValue,
          currentHappy,
          currentGym,
          energyPerTrain,
          currentStat
        )
        
        // 更新对应属性的值
        if (currentStat === 'strength') strengthStats += gain
        else if (currentStat === 'speed') speedStats += gain
        else if (currentStat === 'dexterity') dexterityStats += gain
        else defenseStats += gain
        
        remainingEnergy -= energyPerTrain
        gymCapacityLeft -= energyPerTrain
        currentStatIndex = (currentStatIndex + 1) % 4
      }
      
      if (gymCapacityLeft <= energyPerTrain && currentGym < BS_CONSTANTS.L.length - 1) {
        currentGym++
        gymCapacityLeft = BS_CONSTANTS.W[currentGym]
      } else if (actualTrains === 0) {
        break
      }
    }
  }
  
  // 计算总属性
  let totalStats = strengthStats + speedStats + dexterityStats + defenseStats
  
  // SE增强剂加成 - 重新实现
  const statEnhancers = stats.items?.used?.stat_enhancers || 0
  if (statEnhancers > 0) {
    // 计算总的SE增长潜力
    const originalTotal = totalStats
    const seEnhancedTotal = 0.5 * originalTotal + 0.25 * originalTotal * (1 + 0.85 * (Math.pow(1.01, 0.8 * statEnhancers) - 1)) + 0.25 * originalTotal * (1 + 0.85 * (Math.pow(1.01, 0.2 * statEnhancers) - 1))
    const totalSeGrowth = seEnhancedTotal - originalTotal
    
    // SE分配策略：优先级为力量 → 速度 → 敏捷 → 防御
    const seCapPerStat = 500000000000000  // 500T
    const remainingSeCap = 5000000000000   // 5T
    
    // 原始属性值（SE前）
    const originalStrength = strengthStats
    const originalSpeed = speedStats
    const originalDexterity = dexterityStats
    const originalDefense = defenseStats
    
    let remainingSeGrowth = totalSeGrowth
    
    // 第一优先级：力量
    if (remainingSeGrowth > 0) {
      const maxStrengthGrowth = seCapPerStat - originalStrength
      if (maxStrengthGrowth > 0) {
        const strengthSeGrowth = Math.min(remainingSeGrowth, maxStrengthGrowth)
        strengthStats += strengthSeGrowth
        remainingSeGrowth -= strengthSeGrowth
      }
    }
    
    // 第二优先级：速度
    if (remainingSeGrowth > 0) {
      const maxSpeedGrowth = seCapPerStat - originalSpeed
      if (maxSpeedGrowth > 0) {
        const speedSeGrowth = Math.min(remainingSeGrowth, maxSpeedGrowth)
        speedStats += speedSeGrowth
        remainingSeGrowth -= speedSeGrowth
      }
    }
    
    // 第三优先级：敏捷
    if (remainingSeGrowth > 0) {
      const maxDexterityGrowth = seCapPerStat - originalDexterity
      if (maxDexterityGrowth > 0) {
        const dexteritySeGrowth = Math.min(remainingSeGrowth, maxDexterityGrowth)
        dexterityStats += dexteritySeGrowth
        remainingSeGrowth -= dexteritySeGrowth
      }
    }
    
    // 第四优先级：防御（只能增长5T）
    if (remainingSeGrowth > 0) {
      const maxDefenseGrowth = Math.min(remainingSeCap, remainingSeGrowth)
      defenseStats += maxDefenseGrowth
      remainingSeGrowth -= maxDefenseGrowth
    }
    
    // 更新总属性
    totalStats = strengthStats + speedStats + dexterityStats + defenseStats
  }
  
  return Math.floor(totalStats)
}

// 格式化BS数值显示（k, m, b, t, q）
const formatBSValue = (value) => {
  if (!value || value === 0) return '0'
  
  const absValue = Math.abs(value)
  if (absValue < 1000) return value.toString()
  if (absValue < 1000000) return (value / 1000).toFixed(1) + 'k'
  if (absValue < 1000000000) return (value / 1000000).toFixed(1) + 'm'
  if (absValue < 1000000000000) return (value / 1000000000).toFixed(1) + 'b'
  if (absValue < 1000000000000000) return (value / 1000000000000).toFixed(1) + 't'
  return (value / 1000000000000000).toFixed(1) + 'q'
}

// BS预测算法实现 - 增强错误处理
const calculateBSPrediction = (userProfile, personalStats, criminalRecord) => {
  try {
    console.log(`开始BS计算 - 用户: ${userProfile?.name || 'Unknown'}`)
    
    if (!userProfile || !personalStats) {
      console.warn('BS预测：缺少必要数据', { hasProfile: !!userProfile, hasPersonalStats: !!personalStats })
      return { bs: 0, bsScore: 0, confidence: 'low' }
    }
    
    // 详细检查数据完整性
    const profile = userProfile
    const stats = personalStats
    
    console.log(`用户 ${profile?.name || 'Unknown'} 数据检查:`, {
      age: profile?.age,
      level: profile?.level,
      hasOtherStats: !!stats?.other,
      hasDrugsStats: !!stats?.drugs,
      hasAttackingStats: !!stats?.attacking,
      hasItemsStats: !!stats?.items
    })
    
    // 1. 估算总能量消耗
    const totalEnergy = calculateTotalEnergy(profile, stats)
    console.log(`用户 ${profile?.name || 'Unknown'} 总能量:`, totalEnergy)
    
    if (totalEnergy <= 0) {
      console.warn(`用户 ${profile?.name || 'Unknown'} 总能量为0或负数`)
      return { bs: 1000, bsScore: 63, confidence: 'low' } // 给一个最小默认值
    }
    
    // 2. 模拟健身房锻炼（使用新算法）
    const totalStats = simulateGymTraining(totalEnergy, stats, profile)
    console.log(`用户 ${profile?.name || 'Unknown'} 模拟结果:`, totalStats)
    
    if (totalStats <= 0) {
      console.warn(`用户 ${profile?.name || 'Unknown'} 模拟结果为0或负数`)
      return { bs: 1000, bsScore: 63, confidence: 'low' } // 给一个最小默认值
    }
    
    // 3. 计算BS分数（新的分段计算公式）
    let bsScore
    const tenBillion = 10000000000 // 10b
    
    if (totalStats < tenBillion) {
      // 小于10b：使用原公式
      bsScore = Math.sqrt(totalStats) * 2
      console.log(`用户 ${profile?.name || 'Unknown'} 使用原公式: sqrt(${totalStats}) * 2 = ${bsScore}`)
    } else {
      // 大于等于10b：分段计算
      const tenBillionBsScore = Math.sqrt(tenBillion) * 2 // 10b的BS分 = 200,000
      const ratio = totalStats / tenBillion
      const logRatio = Math.log10(ratio)
      bsScore = tenBillionBsScore * (1 + logRatio)
      console.log(`用户 ${profile?.name || 'Unknown'} 使用分段公式: ${tenBillionBsScore} * (1 + log10(${ratio})) = ${bsScore}`)
    }
    
    console.log(`用户 ${profile?.name || 'Unknown'} 最终结果: 总属性=${totalStats}, BS分=${bsScore}`)
    
    return {
      bs: Math.floor(totalStats),
      bsScore: Math.floor(bsScore),
      confidence: totalEnergy > 1000000 ? 'high' : totalEnergy > 100000 ? 'medium' : 'low'
    }
  } catch (error) {
    console.error(`用户 ${userProfile?.name || 'Unknown'} BS预测计算失败:`, error)
    return { bs: 1000, bsScore: 63, confidence: 'error' } // 出错时给默认值
  }
}

// 计算总能量消耗 - 增强错误处理
const calculateTotalEnergy = (profile, stats) => {
  try {
    const now = Math.floor(Date.now() / 1000)
    const startTimestamp = Math.floor(new Date('2011-11-22').getTime() / 1000)
    
    // 安全获取数值，提供默认值
    const age = profile?.age || 100
    const donatorDays = stats?.other?.donator_days || 0
    
    // 计算捐献者比例
    const m = Math.min(age, (now - startTimestamp) / 86400)
    const donatorPercent = m > 0 ? Math.min(donatorDays / m, 1) : 0
    
    // 估算活跃天数
    const y = 480 + 240 * donatorPercent
    const F = 611255 / y
    const lastActionTime = profile?.last_action?.timestamp || now
    const a = (now - lastActionTime) / 86400
    const ageM = Math.max(1, 21 * (age - a) / 24)
    
    const activityTime = stats?.other?.activity?.time || 0
    const travelTime = stats?.travel?.time_spent || 0
    const N = 3 * (activityTime / 86400) + (travelTime / 86400)
    
    // 药物活跃度计算 - 安全获取数值
    const drugs = stats?.drugs || {}
    const exttaken = drugs.ecstasy || 0
    const victaken = drugs.vicodin || 0
    const kettaken = drugs.ketamine || 0
    const lsdtaken = drugs.lsd || 0
    const opitaken = drugs.opium || 0
    const pcptaken = drugs.pcp || 0
    const shrtaken = drugs.shrooms || 0
    const spetaken = drugs.speed || 0
    const cantaken = drugs.cannabis || 0
    const xantaken = drugs.xanax || 0
    
    const drugEnergy = (
      75 * exttaken +
      210 * victaken +
      52.5 * kettaken +
      425 * lsdtaken +
      215 * opitaken +
      430 * pcptaken +
      209.5 * shrtaken +
      301 * spetaken +
      300 * cantaken +
      420 * xantaken
    )
    
    const S_drugs = drugEnergy / 1440
    
    // 犯罪活跃度计算 - 安全获取数值
    let crimeEnergy = 0
    const criminalRecord = stats?.criminalrecord || {}
    
    if (Object.keys(criminalRecord).length > 0) {
      // 判断是否存在vandalism (D标志)
      const D = (criminalRecord.vandalism || 0) > 0
      
      // 根据D值计算不同的犯罪系数
      let c2, c3, c5, c8, c9, c10, c11, c12
      
      if (D) {
        c2 = 0.1 * (criminalRecord.theft || 0)
        c3 = criminalRecord.counterfeiting || 0
        c5 = 0.65 * (criminalRecord.theft || 0)
        c8 = (criminalRecord.illicitservices || 0) / 2
        c9 = criminalRecord.cybercrime || 0
        c10 = (criminalRecord.illicitservices || 0) / 2
        c11 = criminalRecord.fraud || 0
        c12 = 0.25 * (criminalRecord.theft || 0)
      } else {
        c2 = criminalRecord.other || 0
        c3 = criminalRecord.selling_illegal_products || 0
        c5 = criminalRecord.theft || 0
        c8 = criminalRecord.drug_deals || 0
        c9 = criminalRecord.computer_crimes || 0
        c10 = criminalRecord.murder || 0
        c11 = criminalRecord.fraud_crimes || 0
        c12 = criminalRecord.auto_theft || 0
      }
      
      // 计算犯罪能量
      crimeEnergy = 5 * (
        2 * c2 +
        3 * c3 +
        5 * c5 +
        8 * (c8 / 0.8) +
        9 * (c9 / 0.75) +
        10 * (c10 / 0.75) +
        11 * (c11 / 0.95) +
        12 * (c12 / 0.7)
      )
    }
    
    let n_crimes = crimeEnergy / 1440
    
    // 修正犯罪活跃度
    if (n_crimes < F && n_crimes > 0) {
      const F_corrected = Math.min(F / n_crimes, 3)
      n_crimes *= F_corrected
    }
    
    const estimateActiveDays = Math.min(ageM, Math.max(N, S_drugs, n_crimes))
    
    // 计算各部分能量 - 安全获取数值
    const natureEnergy = y * estimateActiveDays
    const itemEnergy = (
      150 * (stats?.other?.refills?.energy || 0) +
      250 * xantaken +
      50 * lsdtaken +
      20 * (stats?.items?.used?.energy_drinks || 0) +
      150 * (stats?.items?.used?.boosters || 0)
    )
    
    const attacks = stats?.attacking?.attacks || {}
    const expendEnergy = (
      25 * ((attacks.won || 0) + (attacks.stalemate || 0) + (attacks.lost || 0)) +
      25 * (stats?.hospital?.reviving?.revives || 0) +
      5 * (stats?.items?.found?.dump || 0)
    )
    
    const totalEnergy = Math.max(0, natureEnergy + itemEnergy - expendEnergy)
    
    // 如果计算出的能量太低，给一个最小值
    return Math.max(totalEnergy, 1000)
  } catch (error) {
    console.error('计算总能量失败:', error)
    return 10000 // 返回一个默认值
  }
}

// 辅助函数
const getMemberCount = (members) => {
  if (!members) return 0
  if (Array.isArray(members)) return members.length
  return Object.keys(members).length
}

// 格式化睡觉时间段
const formatSleepPeriod = (sleepPeriod) => {
  if (!sleepPeriod) return '无数据'
  
  const startHour = sleepPeriod.start
  const endHour = (sleepPeriod.start + sleepPeriod.duration) % 24
  
  return `${startHour.toString().padStart(2, '0')}:00-${endHour.toString().padStart(2, '0')}:00 (${sleepPeriod.duration}h)`
}

// 格式化活跃时间段
const formatPeakHours = (activeRanges) => {
  if (!activeRanges || activeRanges.length === 0) return '无数据'
  
  // 格式化时间段范围
  const ranges = activeRanges.map(range => {
    if (range.start === range.end) {
      return `${range.start.toString().padStart(2, '0')}:00`
    } else {
      const endHour = (range.end + 1) % 24 // 结束时间+1小时表示区间
      return `${range.start.toString().padStart(2, '0')}:00-${endHour.toString().padStart(2, '0')}:00`
    }
  })
  
  return ranges.join(', ')
}

// 获取四个月前的时间戳
const getFourMonthsAgo = () => {
  const now = new Date()
  now.setMonth(now.getMonth() - 4)
  return Math.floor(now.getTime() / 1000)
}

// 获取四个月前的日期（用于缓存键，更精确到天）
const getFourMonthsAgoDateString = () => {
  const now = new Date()
  now.setMonth(now.getMonth() - 4)
  return now.toISOString().split('T')[0] // YYYY-MM-DD格式
}

// 更新进度
const updateProgress = (step, total, message) => {
  overallProgress.value = Math.round((step / total) * 100)
  progressText.value = `${step}/${total} - ${message}`
}

const updateDetailedProgress = (key, label, step, total, status = 'active') => {
  const existing = detailedProgress.value.find(item => item.key === key)
  const percentage = total > 0 ? Math.round((step / total) * 100) : 0
  const text = `${step}/${total}`
  
  if (existing) {
    existing.percentage = percentage
    existing.text = text
    existing.status = status
  } else {
    detailedProgress.value.push({
      key,
      label,
      percentage,
      text,
      status
    })
  }
}

// 获取帮派基本信息（增加重试机制）
const getFactionInfo = async (factionId, requestQueue) => {
  const cacheKey = getCacheKey('faction', factionId)
  let cached = getCachedData(cacheKey)
  
  if (cached) {
    return cached
  }
  
  statusMessage.value = `Fetching basic information for faction ${factionId}...`
  
  const data = await requestQueue.addRequest(async (apiKey) => {
    return await fetchApiWithRetry(`/faction/${factionId}`, apiKey, {
      context: `帮派 ${factionId} 基本信息`
    })
  })
  
  // 根据实际数据结构提取帮派信息
  const factionInfo = data.basic || data.faction || data
  if (!factionInfo || !factionInfo.name) {
    console.error('帮派数据结构:', data)
    throw new Error(`Failed to fetch faction ${factionId} information: incorrect data format`)
  }
  
  setCachedData(cacheKey, factionInfo, factionInfo.name)
  return factionInfo
}

// 获取帮派成员列表（增加重试机制）
const getFactionMembers = async (factionId, requestQueue) => {
  const cacheKey = getCacheKey('members', factionId)
  let cached = getCachedData(cacheKey)
  
  if (cached) {
    return cached
  }
  
  statusMessage.value = `Fetching member list for faction ${factionId}...`
  
  const data = await requestQueue.addRequest(async (apiKey) => {
    return await fetchApiWithRetry(`/faction/${factionId}/members?striptags=true`, apiKey, {
      context: `帮派 ${factionId} 成员列表`
    })
  })
  
  // 根据实际数据结构提取成员信息
  const members = data.members || data
  if (!members || !Array.isArray(members)) {
    console.error('成员数据结构:', data)
    throw new Error(`Failed to fetch faction ${factionId} member list: incorrect data format`)
  }
  
  // 将成员数组转换为以ID为键的对象，方便后续处理
  const membersObj = {}
  members.forEach(member => {
    membersObj[member.id] = member
  })
  
  setCachedData(cacheKey, membersObj)
  return membersObj
}

// 获取成员个人数据（增加重试机制）
// Fetches personal statistics for a member.
const getMemberPersonalStats = async (memberId, requestQueue, memberIndex, totalMembers, factionId) => {
  const cacheKey = getCacheKey('personalstats', memberId);
  let cached = getCachedData(cacheKey);

  if (cached) {
    updateDetailedProgress(`members_${factionId}`, `Faction ${factionId} member data`, memberIndex, totalMembers);
    return cached;
  }

  updateDetailedProgress(`members_${factionId}`, `Faction ${factionId} member data`, memberIndex, totalMembers);

  try {
    // Check if the request was canceled.
    if (abortController.value?.signal.aborted) {
      throw new Error('Request canceled');
    }

    // Concurrently fetch user profile and personal stats (with retries).
    const [profileData, personalStatsData] = await Promise.all([
      requestQueue.addRequest(async (apiKey) => {
        return await fetchApiWithRetry(`/user/${memberId}`, apiKey, {
          context: `Member ${memberId} basic info`
        });
      }),
      requestQueue.addRequest(async (apiKey) => {
        return await fetchApiWithRetry(`/user/${memberId}/personalstats?cat=all`, apiKey, {
          context: `Member ${memberId} personal stats`
        });
      })
    ]);

    // Check again if the request was canceled.
    if (abortController.value?.signal.aborted) {
      throw new Error('Request canceled');
    }

    // Merge the data.
    const combinedData = {
      profile: profileData.profile || profileData,
      personalstats: personalStatsData.personalstats || personalStatsData,
      criminalrecord: personalStatsData.criminalrecord || (profileData.criminalrecord || {})
    };

    if (!combinedData.personalstats) {
      console.warn(`Personal data for member ${memberId} is empty`);
      return null;
    }

    setCachedData(cacheKey, combinedData);
    return combinedData;
  } catch (error) {
    if (error.message === 'Request canceled') {
      throw error;
    }
    console.error(`Failed to fetch personal data for member ${memberId}:`, error);
    return null;
  }
};

// Fetches faction ranked war data (with retry mechanism).
const getFactionRankedWars = async (factionId, requestQueue) => {
  const fourMonthsAgo = getFourMonthsAgo();
  const cacheKey = getCacheKey('rankedwars', factionId, getFourMonthsAgoDateString());
  let cached = getCachedData(cacheKey);

  if (cached) {
    return cached;
  }

  statusMessage.value = `Fetching RW data for faction ${factionId}...`;

  try {
    // 1. Fetch the basic list of ranked wars (with retries).
    const data = await requestQueue.addRequest(async (apiKey) => {
      return await fetchApiWithRetry(`/faction/${factionId}/rankedwars`, apiKey, {
        context: `Faction ${factionId} RW list`
      });
    });

    // Extract ranked war data.
    const allRankedWars = data.rankedwars || data.rankedwarreports || data || [];
    console.log(`Faction ${factionId} found ${Object.keys(allRankedWars).length} RW records`);
    console.log(`Faction ${factionId} RW basic data structure:`, {
      dataKeys: Object.keys(data),
      rankedwarsKeys: data.rankedwars ? Object.keys(data.rankedwars) : [],
      firstRwId: Object.keys(allRankedWars)[0],
      firstRwData: allRankedWars[Object.keys(allRankedWars)[0]]
    });

    // 2. Filter for ranked wars within the last four months.
    const recentRankedWars = {};
    const currentTime = Math.floor(Date.now() / 1000);

    for (const [warKey, war] of Object.entries(allRankedWars)) {
      if (war.start >= fourMonthsAgo) {
        // Use war.id as the actual war ID.
        recentRankedWars[war.id] = war;
      }
    }

    console.log(`Faction ${factionId} has ${Object.keys(recentRankedWars).length} RW records left after filtering for the last four months`);
    console.log(`Faction ${factionId} real RW ID example:`, Object.keys(recentRankedWars).slice(0, 3));

    // 3. Fetch detailed reports for each ranked war with high concurrency (with retry mechanism).
    const detailedRankedWars = [];
    const warIds = Object.keys(recentRankedWars); // These are now the actual war.id values.
    const apiKeys = getValidApiKeys();

    updateDetailedProgress(`rw_${factionId}`, `Faction ${factionId} RW detailed report`, 0, warIds.length, '');

    if (warIds.length > 0) {
      console.log(`Starting concurrent fetch for ${warIds.length} RW detailed reports, using ${apiKeys.length} API keys`);

      let processedCount = 0;
      const rwQueue = [...warIds]; // Copy the queue.

      // Create concurrent workers, one for each API key.
      const workers = apiKeys.map(async (apiKey, workerIndex) => {
        console.log(`RW worker ${workerIndex + 1} started`);

        while (rwQueue.length > 0) {
          // Check if the request was canceled.
          if (abortController.value?.signal.aborted) {
            throw new Error('Request canceled');
          }

          // Take a ranked war from the queue.
          const warId = rwQueue.shift();
          if (!warId) break;

          try {
            console.log(`RW worker ${workerIndex + 1} started fetching detailed report for RW ${warId}`);
            console.log(`API request URL: /faction/${warId}/rankedwarreport`);
            console.log(`RW ID type and value:`, { warId, type: typeof warId, isString: typeof warId === 'string' });

            // Fetch the detailed ranked war report with retries.
            const reportData = await fetchApiWithRetry(`/faction/${warId}/rankedwarreport`, apiKey, {
              context: `RW ${warId} detailed report`
            });

            if (abortController.value?.signal.aborted) {
              throw new Error('Request canceled');
            }

            console.log(`RW ${warId} detailed report data:`, reportData);

            if (reportData.rankedwarreport) {
              detailedRankedWars.push({
                id: warId, // This is now the actual war.id.
                basicInfo: recentRankedWars[warId],
                report: reportData.rankedwarreport
              });
              console.log(`RW worker ${workerIndex + 1} successfully fetched RW ${warId}`);
            } else if (reportData.war || reportData.factions) {
              // Try to use the data directly.
              detailedRankedWars.push({
                id: warId, // This is now the actual war.id.
                basicInfo: recentRankedWars[warId],
                report: reportData
              });
              console.log(`RW worker ${workerIndex + 1} fetched RW ${warId} using direct format`);
            } else {
              console.warn(`RW ${warId} report data format is abnormal:`, Object.keys(reportData));
              detailedRankedWars.push({
                id: warId, // This is now the actual war.id.
                basicInfo: recentRankedWars[warId],
                report: null
              });
            }

          } catch (reportError) {
            if (reportError.message === 'Request canceled') {
              throw reportError;
            }
            console.error(`RW worker ${workerIndex + 1} failed to fetch detailed report for RW ${warId}:`, reportError);
            detailedRankedWars.push({
              id: warId, // This is now the actual war.id.
              basicInfo: recentRankedWars[warId],
              report: null
            });
          }

          // Update progress.
          processedCount++;
          updateDetailedProgress(`rw_${factionId}`, `Faction ${factionId} RW detailed report`, processedCount, warIds.length, '');

          // Wait interval.
          await new Promise(resolve => setTimeout(resolve, 1200));
        }

        console.log(`RW worker ${workerIndex + 1} finished work`);
      });

      // Wait for all workers to complete.
      await Promise.all(workers);
    }

    updateDetailedProgress(`rw_${factionId}`, `Faction ${factionId} RW detailed report`, warIds.length, warIds.length, 'success');

    setCachedData(cacheKey, detailedRankedWars);
    return detailedRankedWars;
  } catch (error) {
    console.error(`Failed to fetch RW data for faction ${factionId}:`, error);
    return [];
  }
};

// Fetches faction chain data (with retry mechanism).
const getFactionChains = async (factionId, requestQueue, rankedWars = []) => {
  const fourMonthsAgo = getFourMonthsAgo();
  const cacheKey = getCacheKey('chains', factionId, getFourMonthsAgoDateString());
  let cached = getCachedData(cacheKey);

  if (cached) {
    return cached;
  }

  statusMessage.value = `Fetching Chain data for faction ${factionId}...`;

  try {
    // 1. Collect time ranges for all ranked wars.
    const timeRanges = [];
    for (const rwData of rankedWars) {
      if (rwData.report) {
        timeRanges.push({
          start: rwData.report.start,
          end: rwData.report.end,
          warId: rwData.id
        });
      } else if (rwData.basicInfo) {
        // If there is no detailed report, use basic info.
        timeRanges.push({
          start: rwData.basicInfo.start,
          end: rwData.basicInfo.end || (rwData.basicInfo.start + 24 * 3600), // Assuming the war lasts 24 hours.
          warId: rwData.id
        });
      }
    }

    console.log(`Faction ${factionId} needs to fetch Chain data for ${timeRanges.length} RW time periods`);

    // 2. Get all basic Chain lists within four months (with retry).
    const data = await requestQueue.addRequest(async (apiKey) => {
      return await fetchApiWithRetry(`/faction/${factionId}/chains?from=${fourMonthsAgo}`, apiKey, {
        context: `Faction ${factionId} Chain list`
      });
    });

    const allChains = data.chains || data || [];
    console.log(`Faction ${factionId} found ${Object.keys(allChains).length} Chain records`);

    // 3. Filter Chains within the ranked war time range.
    const rwChains = {};
    for (const [chainKey, chain] of Object.entries(allChains)) {
      // Check if the Chain is within any RW time range.
      for (const timeRange of timeRanges) {
        const chainStart = chain.start;
        const chainEnd = chain.end;

        // Determine if Chain time overlaps with RW time.
        if ((chainStart >= timeRange.start && chainStart <= timeRange.end) ||
            (chainEnd >= timeRange.start && chainEnd <= timeRange.end) ||
            (chainStart <= timeRange.start && chainEnd >= timeRange.end)) {

          // Use chain.id as the real Chain ID.
          if (!rwChains[chain.id]) {
            rwChains[chain.id] = {
              ...chain,
              relatedWars: []
            };
          }
          rwChains[chain.id].relatedWars.push(timeRange.warId);
        }
      }
    }

    console.log(`Faction ${factionId} has ${Object.keys(rwChains).length} RW-related Chain records left after filtering`);

    // 4. Fetch detailed reports for each Chain with high concurrency (with retry mechanism).
    const detailedChains = [];
    const chainIds = Object.keys(rwChains); // These are now the actual chain.id values.
    const apiKeys = getValidApiKeys();

    updateDetailedProgress(`chains_${factionId}`, `Faction ${factionId} Chain detailed report`, 0, chainIds.length, '');

    if (chainIds.length > 0) {
      console.log(`Starting concurrent fetch for ${chainIds.length} Chain detailed reports, using ${apiKeys.length} API keys`);

      let processedCount = 0;
      const chainQueue = [...chainIds]; // Copy the queue.

      // Create concurrent workers, one for each API key.
      const workers = apiKeys.map(async (apiKey, workerIndex) => {
        console.log(`Chain worker ${workerIndex + 1} started`);

        while (chainQueue.length > 0) {
          // Check if the request was canceled.
          if (abortController.value?.signal.aborted) {
            throw new Error('Request canceled');
          }

          // Take a Chain from the queue.
          const chainId = chainQueue.shift();
          if (!chainId) break;

          try {
            console.log(`Chain worker ${workerIndex + 1} started fetching detailed report for Chain ${chainId}`);

            // Fetch the detailed Chain report with retries.
            const reportData = await fetchApiWithRetry(`/faction/${chainId}/chainreport`, apiKey, {
              context: `Chain ${chainId} detailed report`
            });

            if (abortController.value?.signal.aborted) {
              throw new Error('Request canceled');
            }

            if (reportData.chainreport && reportData.chainreport.attackers) {
              detailedChains.push({
                id: chainId, // This is now the actual chain.id.
                basicInfo: rwChains[chainId],
                report: reportData.chainreport,
                relatedWars: rwChains[chainId].relatedWars
              });
              console.log(`Chain worker ${workerIndex + 1} successfully fetched Chain ${chainId}, number of attackers: ${reportData.chainreport.attackers.length}`);
            } else if (reportData.attackers) {
              detailedChains.push({
                id: chainId, // This is now the actual chain.id.
                basicInfo: rwChains[chainId],
                report: reportData,
                relatedWars: rwChains[chainId].relatedWars
              });
              console.log(`Chain worker ${workerIndex + 1} fetched Chain ${chainId} (direct format), number of attackers: ${reportData.attackers.length}`);
            } else {
              console.warn(`Chain ${chainId} report data structure is abnormal`);
              detailedChains.push({
                id: chainId, // This is now the actual chain.id.
                basicInfo: rwChains[chainId],
                report: null,
                relatedWars: rwChains[chainId].relatedWars
              });
            }

          } catch (reportError) {
            if (reportError.message === 'Request canceled') {
              throw reportError;
            }
            console.error(`Chain worker ${workerIndex + 1} failed to fetch detailed report for Chain ${chainId}:`, reportError);
            detailedChains.push({
              id: chainId, // This is now the actual chain.id.
              basicInfo: rwChains[chainId],
              report: null,
              relatedWars: rwChains[chainId].relatedWars
            });
          }

          // Update progress.
          processedCount++;
          updateDetailedProgress(`chains_${factionId}`, `Faction ${factionId} Chain detailed report`, processedCount, chainIds.length, '');

          // Wait interval.
          await new Promise(resolve => setTimeout(resolve, 1200));
        }

        console.log(`Chain worker ${workerIndex + 1} finished work`);
      });

      // Wait for all workers to complete.
      await Promise.all(workers);
    }

    updateDetailedProgress(`chains_${factionId}`, `Faction ${factionId} Chain detailed report`, chainIds.length, chainIds.length, 'success');

    setCachedData(cacheKey, detailedChains);
    return detailedChains;
  } catch (error) {
    console.error(`Failed to fetch Chain data for faction ${factionId}:`, error);
    return [];
  }
};

// Analyzes attack counts and active times in Chain data.
const analyzeChainActivity = (chains) => {
  let totalAttacks = 0;
  let hosAttacks = 0;
  let revengeAttacks = 0;
  const timeZoneHours = new Array(24).fill(0);
  const fourMonthsAgo = getFourMonthsAgo();
  const oneMonthAgo = Math.floor(Date.now() / 1000) - (30 * 24 * 3600);

  let recentTotalAttacks = 0; // Last month.

  console.log(`Analyzing overall Chain activity - Total chains: ${chains.length}`);

  chains.forEach((chainData, chainIndex) => {
    if (chainData.report && chainData.report.attackers) {
      // Using new data structure: chainreport.attackers array.
      chainData.report.attackers.forEach(attacker => {
        if (attacker.attacks) {
          const attacks = attacker.attacks;
          totalAttacks += attacks.total || 0;
          hosAttacks += attacks.hospitalize || 0;
          revengeAttacks += attacks.retaliations || 0;

          // Check if it's the last month (using Chain start time as an approximation).
          if (chainData.report.start >= oneMonthAgo) {
            recentTotalAttacks += attacks.total || 0;
          }

          // Timezone analysis - improved algorithm.
          const totalAttackCount = attacks.total || 0;
          if (totalAttackCount > 0) {
            const chainStart = new Date(chainData.report.start * 1000);
            const chainEnd = new Date(chainData.report.end * 1000);
            const chainDuration = (chainData.report.end - chainData.report.start) / 3600; // in hours

            // Estimate attack distribution based on Chain duration and number of attacks.
            for (let i = 0; i < totalAttackCount; i++) {
              // Evenly distribute attack times within the Chain duration.
              const attackOffset = (chainDuration * i / totalAttackCount); // Relative time of the attack in the Chain (hours).
              const attackTime = new Date(chainStart.getTime() + attackOffset * 3600 * 1000);
              const beijingHour = (attackTime.getUTCHours()) % 24; // Changed from +8 to UTC (no offset).
              timeZoneHours[beijingHour]++;
            }
          }
        }
      });
    }
  });

  // Calculate active time periods.
  const peakHours = [];
  const maxActivity = Math.max(...timeZoneHours);
  const threshold = maxActivity * 0.7; // Activity above 70% is considered an active period.

  for (let hour = 0; hour < 24; hour++) {
    if (timeZoneHours[hour] >= threshold && timeZoneHours[hour] > 0) {
      peakHours.push(hour);
    }
  }

  const result = {
    totalAttacks,
    recentAttacks: recentTotalAttacks,
    hosPercentage: totalAttacks > 0 ? (hosAttacks / totalAttacks * 100) : 0,
    revengePercentage: totalAttacks > 0 ? (revengeAttacks / totalAttacks * 100) : 0,
    peakHours,
    timeZoneDistribution: timeZoneHours
  };

  console.log(`Overall Chain activity: Total attacks ${totalAttacks}, Recent attacks ${recentTotalAttacks}, HOS ${result.hosPercentage.toFixed(1)}%`);
  return result;
};


// Calculate activity score (redesigned, mainly based on attack count).
const calculateActivityScore = (chainActivity, bsScore) => {
  // New activity calculation: mainly based on attack count, no upper limit.
  const fourMonthWeight = 0.6; // Four-month attack count weight.
  const oneMonthWeight = 0.3;  // One-month attack count weight.
  const timeRangeWeight = 0.1; // Active time period weight.

  // Use attack count directly, no normalization limit.
  const fourMonthScore = chainActivity.fourMonthAttacks * fourMonthWeight;
  const oneMonthScore = chainActivity.oneMonthAttacks * oneMonthWeight;

  // Bonus for diversity of active time periods (up to 20 points).
  const timeRangeBonus = chainActivity.peakHours.length > 0 ?
    Math.min(chainActivity.peakHours.length * 2.5, 20) : 0;

  const finalScore = fourMonthScore + oneMonthScore + timeRangeBonus;

  console.log(`Activity score calculation: Four-month attacks=${chainActivity.fourMonthAttacks}*${fourMonthWeight}=${fourMonthScore}, One-month attacks=${chainActivity.oneMonthAttacks}*${oneMonthWeight}=${oneMonthScore}, Time diversity=${timeRangeBonus}, Final score=${finalScore}`);

  return Math.max(0, finalScore);
};

// Calculate combat power score (new).
const calculateCombatPowerScore = (memberData) => {
  const {
    estimatedBS,
    bsScore,
    fourMonthAttacks,
    oneMonthAttacks,
    hosPercentage,
    revengePercentage,
    peakHours,
    activityScore
  } = memberData;

  // New weight distribution (excluding BS).
  const weights = {
    activity: 0.75,       // Activity weight 75%
    attackQuality: 0.15,  // Attack quality weight 15%
    consistency: 0.10,    // Consistency weight 10%
    timeRange: 0.0        // Time coverage weight 0%
  };

  // 1. Activity score component.
  const activityComponent = activityScore * weights.activity;

  // 2. Attack quality score (HOS percentage and attack intensity).
  const hosBonus = hosPercentage * 2; // 2 points for every 1% of HOS.
  const attackIntensity = fourMonthAttacks > 0 ? Math.min(fourMonthAttacks / 10, 50) : 0; // 1 point per 10 attacks, max 50.
  const qualityComponent = (hosBonus + attackIntensity) * weights.attackQuality;

  // 3. Consistency score (performance in the last month).
  const consistencyRatio = fourMonthAttacks > 0 ? oneMonthAttacks / (fourMonthAttacks / 4) : 0;
  const consistencyComponent = Math.min(consistencyRatio * 100, 150) * weights.consistency;

  // 4. Time coverage score.
  const timeRangeComponent = (peakHours.length * 10) * weights.timeRange;

  // Calculate base score (excluding BS).
  const baseScore = activityComponent + qualityComponent + consistencyComponent + timeRangeComponent;

  // New formula: (baseScore / 1000) * bsScore.
  const finalScore = (baseScore / 1000) * bsScore;

  console.log(`Combat Power Score Calculation - Activity:${activityComponent.toFixed(1)}, Quality:${qualityComponent.toFixed(1)}, Consistency:${consistencyComponent.toFixed(1)}, Time:${timeRangeComponent.toFixed(1)}, Base Score:${baseScore.toFixed(1)}, BS Score:${bsScore}, Final Score:${finalScore.toFixed(1)}`);

  return {
    totalScore: Math.round(finalScore),
    components: {
      activity: Math.round(activityComponent),
      quality: Math.round(qualityComponent),
      consistency: Math.round(consistencyComponent),
      timeRange: Math.round(timeRangeComponent),
      baseScore: Math.round(baseScore),
      bsMultiplier: bsScore
    }
  };
};

// Main faction strength analysis function.
const analyzeFactionStrength = (factionData) => {
  console.log('Starting faction strength analysis:', factionData);

  if (!factionData.members || !factionData.personalStats || !factionData.chains) {
    console.warn('Faction strength analysis: missing necessary data');
    return null;
  }

  // Analyze overall Chain activity.
  const overallActivity = analyzeChainActivity(factionData.chains);

  // Analyze data for each member.
  const memberAnalysis = analyzeMemberData(
    factionData.members,
    factionData.personalStats,
    factionData.chains
  );

  // Calculate total faction power score (using the new combat power score).
  const totalCombatPower = memberAnalysis.reduce((sum, member) => sum + member.combatPowerScore, 0);
  const averageCombatPower = memberAnalysis.length > 0 ? totalCombatPower / memberAnalysis.length : 0;

  // Calculate total BS and average BS.
  const totalBS = memberAnalysis.reduce((sum, member) => sum + member.estimatedBS, 0);
  const averageBS = memberAnalysis.length > 0 ? totalBS / memberAnalysis.length : 0;

  // Calculate total activity score and average activity score.
  const totalActivityScore = memberAnalysis.reduce((sum, member) => sum + member.activityScore, 0);
  const averageActivityScore = memberAnalysis.length > 0 ? totalActivityScore / memberAnalysis.length : 0;

  return {
    name: factionData.name,
    memberCount: memberAnalysis.length,
    memberAnalysis,
    overallActivity,
    totalActivityScore,
    averageActivityScore,
    totalBS,
    averageBS,
    totalCombatPower,       // New
    averageCombatPower,     // New
    averageAttacksPerMonth: memberAnalysis.length > 0
      ? memberAnalysis.reduce((sum, m) => sum + m.oneMonthAttacks, 0) / memberAnalysis.length
      : 0,
    averageAttacksFourMonth: memberAnalysis.length > 0
      ? memberAnalysis.reduce((sum, m) => sum + m.fourMonthAttacks, 0) / memberAnalysis.length
      : 0
  };
};


// 计算特定时间段的帮派有效实力
const calculateFactionStrengthAtHour = (factionAnalysis, hour) => {
  if (!factionAnalysis || !factionAnalysis.memberAnalysis) {
    return {
      effectiveCombatPower: 0,
      effectiveActivityScore: 0,
      activeMemberCount: 0,
      memberCount: 0
    }
  }
  
  let totalEffectiveCombatPower = 0
  let totalEffectiveActivityScore = 0
  let activeMemberCount = 0
  const totalMemberCount = factionAnalysis.memberAnalysis.length
  
  factionAnalysis.memberAnalysis.forEach(member => {
    // 判断该成员在这个时间段是否活跃（基于睡觉时间算法）
    const isActiveAtHour = member.peakHours.includes(hour)
    
    if (isActiveAtHour) {
      // 活跃时间：100%战力
      totalEffectiveCombatPower += member.combatPowerScore
      totalEffectiveActivityScore += member.activityScore
      activeMemberCount++
    } else {
      // 睡觉时间：30%战力
      totalEffectiveCombatPower += member.combatPowerScore * 0.3
      totalEffectiveActivityScore += member.activityScore * 0.3
      activeMemberCount += 0.3
    }
  })
  
  return {
    effectiveCombatPower: totalMemberCount > 0 ? totalEffectiveCombatPower / totalMemberCount : 0,
    effectiveActivityScore: totalMemberCount > 0 ? totalEffectiveActivityScore / totalMemberCount : 0,
    activeMemberCount: Math.round(activeMemberCount * 10) / 10, // 保留一位小数
    memberCount: totalMemberCount
  }
}

// 计算24小时胜率表
const calculate24HourWinRates = (faction1Analysis, faction2Analysis) => {
  const hourlyWinRates = []
  
  for (let hour = 0; hour < 24; hour++) {
    // 计算该时间段两帮的有效实力
    const faction1HourStrength = calculateFactionStrengthAtHour(faction1Analysis, hour)
    const faction2HourStrength = calculateFactionStrengthAtHour(faction2Analysis, hour)
    
    // 使用有效实力计算该时间段的胜率
    const hourlyPrediction = predictHourlyWinRate(
      faction1Analysis.name,
      faction2Analysis.name,
      faction1HourStrength,
      faction2HourStrength
    )
    
    hourlyWinRates.push({
      hour,
      timeDisplay: `${hour.toString().padStart(2, '0')}:00`,
      faction1Strength: faction1HourStrength,
      faction2Strength: faction2HourStrength,
      faction1WinRate: hourlyPrediction.faction1WinRate,
      faction2WinRate: hourlyPrediction.faction2WinRate,
      faction1Score: hourlyPrediction.faction1Score,
      faction2Score: hourlyPrediction.faction2Score
    })
  }
  
  return hourlyWinRates
}

// 计算特定时间的胜率
const predictHourlyWinRate = (faction1Name, faction2Name, faction1Strength, faction2Strength) => {
  // 权重分配（与主预测相同）
  const combatPowerWeight = 0.7
  const activityWeight = 0.2
  const memberCountWeight = 0.1
  
  // 计算有效评分
  const faction1Score = (
    (faction1Strength.effectiveCombatPower / 1000) * combatPowerWeight +
    (faction1Strength.effectiveActivityScore / 100) * activityWeight +
    (faction1Strength.activeMemberCount / 50) * memberCountWeight
  ) * 100
  
  const faction2Score = (
    (faction2Strength.effectiveCombatPower / 1000) * combatPowerWeight +
    (faction2Strength.effectiveActivityScore / 100) * activityWeight +
    (faction2Strength.activeMemberCount / 50) * memberCountWeight
  ) * 100
  
  // 计算胜率（与主预测逻辑相同）
  let faction1WinRate, faction2WinRate
  
  if (faction1Score === 0 && faction2Score === 0) {
    faction1WinRate = 50
    faction2WinRate = 50
  } else if (faction2Score === 0) {
    faction1WinRate = 100
    faction2WinRate = 0
  } else if (faction1Score === 0) {
    faction1WinRate = 0
    faction2WinRate = 100
  } else {
    const scoreDiff = faction1Score - faction2Score
    const avgScore = (faction1Score + faction2Score) / 2
    const normalizedDiff = scoreDiff / avgScore * 8
    const sigmoidValue = 1 / (1 + Math.exp(-normalizedDiff))
    
    faction1WinRate = Math.round(sigmoidValue * 100)
    faction2WinRate = 100 - faction1WinRate
    
    faction1WinRate = Math.max(0, Math.min(100, faction1WinRate))
    faction2WinRate = Math.max(0, Math.min(100, faction2WinRate))
  }
  
  return {
    faction1WinRate,
    faction2WinRate,
    faction1Score,
    faction2Score
  }
}

// 预测PVP胜率（重新设计，增加24小时分析）
const predictPVPWinRate = (faction1Analysis, faction2Analysis) => {
  if (!faction1Analysis || !faction2Analysis) {
    return { faction1WinRate: 50, faction2WinRate: 50, analysis: '数据不足，无法预测' }
  }
  
  // 计算24小时胜率表
  const hourlyWinRates = calculate24HourWinRates(faction1Analysis, faction2Analysis)
  
  // 计算平均胜率
  const avgFaction1WinRate = Math.round(
    hourlyWinRates.reduce((sum, hour) => sum + hour.faction1WinRate, 0) / 24
  )
  const avgFaction2WinRate = 100 - avgFaction1WinRate
  
  console.log(`24小时平均胜率 - 帮派1:${avgFaction1WinRate}%, 帮派2:${avgFaction2WinRate}%`)
  
  // 生成格式化的分析说明
  const analysisData = {
    faction1: {
      name: faction1Analysis.name,
      averageBS: formatBSValue(Math.round(faction1Analysis.averageBS)),
      activityScore: Math.round(faction1Analysis.averageActivityScore),
      combatPowerScore: Math.round(faction1Analysis.averageCombatPower),
      memberCount: faction1Analysis.memberCount,
      score: Math.round(hourlyWinRates.reduce((sum, hour) => sum + hour.faction1Score, 0) / 24)
    },
    faction2: {
      name: faction2Analysis.name,
      averageBS: formatBSValue(Math.round(faction2Analysis.averageBS)),
      activityScore: Math.round(faction2Analysis.averageActivityScore),
      combatPowerScore: Math.round(faction2Analysis.averageCombatPower),
      memberCount: faction2Analysis.memberCount,
      score: Math.round(hourlyWinRates.reduce((sum, hour) => sum + hour.faction2Score, 0) / 24)
    }
  }
  
  return {
    faction1WinRate: avgFaction1WinRate,
    faction2WinRate: avgFaction2WinRate,
    analysisData: analysisData,
    faction1Score: analysisData.faction1.score,
    faction2Score: analysisData.faction2.score,
    hourlyWinRates: hourlyWinRates // 新增：24小时详细数据
  }
}

// 主要的数据获取函数
const fetchAllData = async () => {
  const apiKeys = getValidApiKeys()
  const requestQueue = new ApiRequestQueue(apiKeys)
  
  abortController.value = new AbortController()
  
  try {
    // First, count the available cached data
    console.log('Checking for available cached data...')
    const faction1Id = form.faction1Id
    const faction2Id = form.faction2Id
    const dateString = getFourMonthsAgoDateString()
    
    const cacheStats = {
      faction1: {
        info: !!getCachedData(getCacheKey('faction', faction1Id)),
        members: !!getCachedData(getCacheKey('members', faction1Id)),
        rankedwars: !!getCachedData(getCacheKey('rankedwars', faction1Id, dateString)),
        chains: !!getCachedData(getCacheKey('chains', faction1Id, dateString))
      },
      faction2: {
        info: !!getCachedData(getCacheKey('faction', faction2Id)),
        members: !!getCachedData(getCacheKey('members', faction2Id)),
        rankedwars: !!getCachedData(getCacheKey('rankedwars', faction2Id, dateString)),
        chains: !!getCachedData(getCacheKey('chains', faction2Id, dateString))
      }
    }
    
    const totalCacheableItems = 8 // 4 data items for each of the two factions
    const cachedItems = Object.values(cacheStats.faction1).filter(Boolean).length + 
                       Object.values(cacheStats.faction2).filter(Boolean).length
    const cacheHitRateBasic = Math.round((cachedItems / totalCacheableItems) * 100)
    
    console.log(`Cache status: ${cachedItems}/${totalCacheableItems} basic data items cached (${cacheHitRateBasic}%)`)
    statusMessage.value = `Starting data retrieval... (${cachedItems}/${totalCacheableItems} basic data items cached)`
    
    // Total steps calculation (this is dynamic because the number of RW and Chains is unknown)
    let totalSteps = 6 // Get basic information
    let currentStep = 0
    
    // Fetching faction basic info...
    updateProgress(++currentStep, totalSteps, 'Fetching faction basic info...')
    console.log('Starting to fetch faction basic info...')
    
    // Check if cancelled
    if (abortController.value?.signal.aborted) {
      throw new Error('Request cancelled')
    }
    
    const [faction1Info, faction2Info] = await Promise.all([
      getFactionInfo(form.faction1Id, requestQueue),
      getFactionInfo(form.faction2Id, requestQueue)
    ])
    console.log('Faction basic info fetch complete:', { faction1Info, faction2Info })
    
    // Fetching member list...
    updateProgress(++currentStep, totalSteps, 'Fetching member list...')
    console.log('Starting to fetch member list...')
    
    // Check if cancelled
    if (abortController.value?.signal.aborted) {
      throw new Error('Request cancelled')
    }
    
    const [faction1Members, faction2Members] = await Promise.all([
      getFactionMembers(form.faction1Id, requestQueue),
      getFactionMembers(form.faction2Id, requestQueue)
    ])
    console.log('Member list fetch complete:', {
      faction1MemberCount: getMemberCount(faction1Members),
      faction2MemberCount: getMemberCount(faction2Members)
    })
    
    // Fetching RW data...
    updateProgress(++currentStep, totalSteps, 'Fetching RW data...')
    console.log('Starting to fetch RW data...')
    
    // Check if cancelled
    if (abortController.value?.signal.aborted) {
      throw new Error('Request cancelled')
    }
    
    const [faction1RankedWars, faction2RankedWars] = await Promise.all([
      getFactionRankedWars(form.faction1Id, requestQueue),
      getFactionRankedWars(form.faction2Id, requestQueue)
    ])
    console.log('RW data fetch complete:', {
      faction1RwCount: faction1RankedWars.length,
      faction2RwCount: faction2RankedWars.length
    })
    
    // Fetching Chain data...
    updateProgress(++currentStep, totalSteps, 'Fetching Chain data...')
    console.log('Starting to fetch Chain data...')
    
    // Check if cancelled
    if (abortController.value?.signal.aborted) {
      throw new Error('Request cancelled')
    }
    
    const [faction1Chains, faction2Chains] = await Promise.all([
      getFactionChains(form.faction1Id, requestQueue, faction1RankedWars),
      getFactionChains(form.faction2Id, requestQueue, faction2RankedWars)
    ])
    console.log('Chain data fetch complete:', {
      faction1ChainCount: faction1Chains.length,
      faction2ChainCount: faction2Chains.length
    })
    
    // Recalculate total steps (including personal data for all members)
    const totalMembers = getMemberCount(faction1Members) + getMemberCount(faction2Members)
    totalSteps = 6 + totalMembers
    
    // Fetching member personal data...
    updateProgress(++currentStep, totalSteps, 'Fetching member personal data...')
    statusMessage.value = 'Fetching member personal data...'
    console.log('Starting to fetch member personal data...')
    
    const faction1PersonalStats = {}
    const faction2PersonalStats = {}
    
    // Prepare all members to be fetched
    const allMembers = [
      ...Object.keys(faction1Members).map(id => ({ id, factionId: form.faction1Id, faction: 'faction1' })),
      ...Object.keys(faction2Members).map(id => ({ id, factionId: form.faction2Id, faction: 'faction2' }))
    ]
    
    console.log(`A total of ${allMembers.length} members' data needs to be fetched, using ${apiKeys.length} API keys`)
    
    // Implement true concurrency: each API key processes one member at the same time
    let processedCount = 0
    let successCount = 0
    let cacheHitCount = 0 // Cache hit count
    let retryCount = 0 // Retry count
    const memberQueue = [...allMembers] // Copy the queue
    
    // Create concurrent workers, one for each API key
    const workers = apiKeys.map(async (apiKey, workerIndex) => {
      console.log(`Worker ${workerIndex + 1} starts working, using API key: ${apiKey.substring(0, 8)}...`)
      
      while (memberQueue.length > 0) {
        // Check if cancelled
        if (abortController.value?.signal.aborted) {
          throw new Error('Request cancelled')
        }
        
        // Dequeue a member
        const member = memberQueue.shift()
        if (!member) break
        
        // Declare the cached variable to ensure it can be accessed throughout the loop
        let cached = false
        
        try {
          console.log(`Worker ${workerIndex + 1} starts fetching data for member ${member.id}`)
          
          // First, check the cache
          const cacheKey = getCacheKey('personalstats', member.id)
          const cachedData = getCachedData(cacheKey)
          
          let combinedData
          if (cachedData) {
            console.log(`Worker ${workerIndex + 1} fetched data for member ${member.id} from cache`)
            combinedData = cachedData
            cached = true
            cacheHitCount++
          } else {
            // Not in cache, initiate API request (with retry mechanism)
            const [profileData, personalStatsData] = await Promise.all([
              fetchApiWithRetry(`/user/${member.id}`, apiKey, {
                context: `Member ${member.id} basic info`
              }),
              fetchApiWithRetry(`/user/${member.id}/personalstats?cat=all`, apiKey, {
                context: `Member ${member.id} personal stats`
              })
            ])
            
            // Check if cancelled
            if (abortController.value?.signal.aborted) {
              throw new Error('Request cancelled')
            }
            
            // Merge data
            combinedData = {
              profile: profileData.profile || profileData,
              personalstats: personalStatsData.personalstats || personalStatsData,
              criminalrecord: personalStatsData.criminalrecord || (profileData.criminalrecord || {})
            }
            
            // Set cache
            setCachedData(cacheKey, combinedData)
            console.log(`Worker ${workerIndex + 1} fetched and cached data for member ${member.id}`)
            cached = false
          }
          
          if (combinedData.personalstats) {
            // Store in the corresponding faction data
            if (member.faction === 'faction1') {
              faction1PersonalStats[member.id] = combinedData
            } else {
              faction2PersonalStats[member.id] = combinedData
            }
            
            successCount++
            console.log(`Worker ${workerIndex + 1} successfully processed data for member ${member.id}`)
          } else {
            console.warn(`Worker ${workerIndex + 1} fetched empty data for member ${member.id}`)
          }
          
        } catch (error) {
          if (error.message === 'Request cancelled') {
            throw error
          }
          console.error(`Worker ${workerIndex + 1} failed to fetch data for member ${member.id}:`, error)
          
          // If the error includes retry information, increment the retry count
          if (error.message.includes('retrying')) {
            retryCount++
          }
        }
        
        // Update progress
        processedCount++
        currentStep++
        const cacheHitRate = processedCount > 0 ? Math.round((cacheHitCount / processedCount) * 100) : 0
        const retryInfo = retryCount > 0 ? `, retries: ${retryCount}` : ''
        statusMessage.value = `Fetching member personal data... (${processedCount}/${allMembers.length}, cache hit rate: ${cacheHitRate}%${retryInfo})`
        updateProgress(currentStep, totalSteps, `Processed ${processedCount}/${allMembers.length} members, successfully fetched ${successCount}, cache hits ${cacheHitCount}${retryInfo}`)
        updateDetailedProgress(`members_all`, `All Members Data`, processedCount, allMembers.length)
        
        // If the data is fetched from the cache, no need to wait
        if (!cached) {
          // Wait a short time after each request to avoid hitting API limits
          await new Promise(resolve => setTimeout(resolve, 1200)) // 50 times/minute = 1.2 second interval
        }
      }
      
      console.log(`Worker ${workerIndex + 1} finished its job`)
    })
    
    // Wait for all workers to complete
    await Promise.all(workers)
    
    console.log(`Personal data fetching complete, successfully fetched data for ${successCount} members, processed ${processedCount} members in total`)
    
    // Data collection complete
    const finalCacheHitRate = processedCount > 0 ? Math.round((cacheHitCount / processedCount) * 100) : 0
    updateProgress(totalSteps, totalSteps, `Data fetch complete! Processed ${processedCount} members, success ${successCount}, cache hit rate ${finalCacheHitRate}%`)
    statusMessage.value = 'Data retrieval completed, analyzing faction strength...'
    
    // Performing faction strength analysis
    statusMessage.value = 'Analyzing faction strength...'
    console.log('Starting faction strength analysis...')
    
    const faction1Analysis = analyzeFactionStrength({
      name: faction1Info.name,
      members: faction1Members,
      personalStats: faction1PersonalStats,
      chains: faction1Chains
    })
    
    const faction2Analysis = analyzeFactionStrength({
      name: faction2Info.name,
      members: faction2Members,
      personalStats: faction2PersonalStats,
      chains: faction2Chains
    })
    
    // Predict PVP win rate
    const winRatePrediction = predictPVPWinRate(faction1Analysis, faction2Analysis)
    
    console.log('Faction strength analysis complete:', { faction1Analysis, faction2Analysis, winRatePrediction })
    
    // Build the result object
    comparisonResult.value = {
      faction1: {
        info: faction1Info,
        name: faction1Info.name,
        members: faction1Members,
        personalStats: faction1PersonalStats,
        rankedWars: faction1RankedWars,
        chains: faction1Chains
      },
      faction2: {
        info: faction2Info,
        name: faction2Info.name,
        members: faction2Members,
        personalStats: faction2PersonalStats,
        rankedWars: faction2RankedWars,
        chains: faction2Chains
      },
      rwDataCount: faction1RankedWars.length + faction2RankedWars.length,
      chainDataCount: faction1Chains.length + faction2Chains.length,
      detailedRwCount: faction1RankedWars.filter(rw => rw.report).length + faction2RankedWars.filter(rw => rw.report).length,
      detailedChainCount: faction1Chains.filter(chain => chain.report).length + faction2Chains.filter(chain => chain.report).length,
      personalStatsCount: Object.values(faction1PersonalStats).filter(stats => stats).length + Object.values(faction2PersonalStats).filter(stats => stats).length,
      // Add strength analysis results
      faction1Analysis,
      faction2Analysis,
      winRatePrediction
    }
    
    statusMessage.value = 'All data fetching complete!'
    statusType.value = 'success'
    
  } catch (error) {
    if (error.message === 'Request cancelled') {
      statusMessage.value = 'Data fetching cancelled'
      statusType.value = 'warning'
    } else {
      console.error('Data fetching failed:', error)
      statusMessage.value = `Data fetching failed: ${error.message}`
      statusType.value = 'error'
    }
  }
}

// Start comparison analysis
const startComparison = async () => {
  if (!canStartComparison.value) {
    ElMessage.error('Please fill in the faction IDs and API keys')
    return
  }
  
  // Validate faction ID format
  const faction1Id = form.faction1Id.trim()
  const faction2Id = form.faction2Id.trim()
  
  if (!/^\d+$/.test(faction1Id) || !/^\d+$/.test(faction2Id)) {
    ElMessage.error('Faction ID must be a number')
    return
  }
  
  if (faction1Id === faction2Id) {
    ElMessage.error('The two faction IDs cannot be the same')
    return
  }
  
  loading.value = true
  statusMessage.value = 'Starting to fetch data...'
  statusType.value = 'info'
  overallProgress.value = 0
  progressText.value = ''
  detailedProgress.value = []
  comparisonResult.value = null
  
  // Update API key pool
  apiKeyPool.value = getValidApiKeys()
  
  console.log(`Starting faction comparison analysis: ${faction1Id} vs ${faction2Id}`)
  console.log(`Using ${apiKeyPool.value.length} API keys`)
  
  await fetchAllData()
  
  loading.value = false
}

// Stop analysis
const stopComparison = () => {
  console.log('User requested to stop analysis')
  
  if (abortController.value) {
    abortController.value.abort()
  }
  
  // Immediately stop the loading state
  loading.value = false
  
  // Reset progress-related states
  overallProgress.value = 0
  progressText.value = ''
  detailedProgress.value = []
  
  // Set cancellation status message
  statusMessage.value = 'Analysis cancelled'
  statusType.value = 'warning'
  
  console.log('Analysis stopped')
}

// Initialize on component mount
onMounted(() => {
  updateCacheInfo()
})

// Analyze individual member data
const analyzeMemberData = (members, personalStats, chains) => {
  const memberAnalysis = []
  
  console.log(`Starting member data analysis - Members: ${Object.keys(members).length}, Personal Stats: ${Object.keys(personalStats).length}`)
  
  // First pass: collect sleep times for all members with data
  const validSleepPeriods = []
  
  Object.entries(members).forEach(([memberId, member]) => {
    const memberData = personalStats[memberId]
    if (!memberData || !memberData.personalstats) {
      return
    }
    
    // Analyze the member's activity in Chains
    const memberChainActivity = analyzeMemberChainActivity(memberId, chains, member.name)
    
    // If the member has attack data, collect their sleep period
    if (memberChainActivity.fourMonthAttacks > 0 && memberChainActivity.sleepPeriod) {
      validSleepPeriods.push(memberChainActivity.sleepPeriod)
    }
  })
  
  // Calculate the faction's average sleep period
  const factionSleepPeriod = calculateFactionAverageSleepPeriod(validSleepPeriods)
  
  // Second pass: assign sleep times and calculate strength for all members
  Object.entries(members).forEach(([memberId, member]) => {
    const memberData = personalStats[memberId]
    if (!memberData || !memberData.personalstats) {
      console.warn(`Member ${member.name} is missing personal data`)
      return
    }
    
    // Calculate BS prediction
    const bsPrediction = calculateBSPrediction(
      memberData.profile || {
        name: member.name,
        age: member.days_in_faction || 100,
        level: member.level,
        rank: member.rank || 'Average',
        last_action: { timestamp: Math.floor(Date.now() / 1000) - 3600 },
        networth: memberData.personalstats?.networth || 0
      },
      memberData.personalstats,
      memberData.personalstats?.crimes || {}
    )
    
    // Analyze the member's activity in Chains
    const memberChainActivity = analyzeMemberChainActivity(memberId, chains, member.name)
    
    // If member has no data, use the faction's average sleep period
    if (memberChainActivity.fourMonthAttacks === 0) {
      console.log(`Member ${member.name} has no attack data, using faction average sleep period`)
      // Recalculate active ranges using faction average sleep period, but set to 10 hours
      const estimatedSleepPeriod = {
        start: factionSleepPeriod.start,
        duration: 10 // Fixed 10-hour sleep time for members with no data
      }
      const activeRanges = calculateActiveRanges(estimatedSleepPeriod)
      
      memberChainActivity.sleepPeriod = estimatedSleepPeriod
      memberChainActivity.activeRanges = activeRanges.ranges
      memberChainActivity.peakHours = activeRanges.activeHours
    }
    
    // Calculate activity score (new algorithm)
    const activityScore = calculateActivityScore(memberChainActivity, bsPrediction.bsScore)
    
    // Prepare member basic information
    const memberInfo = {
      id: memberId,
      name: member.name,
      level: member.level,
      rank: member.rank || 'Unknown',
      estimatedBS: bsPrediction.bs,
      bsScore: bsPrediction.bsScore,
      confidence: bsPrediction.confidence,
      fourMonthAttacks: memberChainActivity.fourMonthAttacks,
      oneMonthAttacks: memberChainActivity.oneMonthAttacks,
      hosPercentage: memberChainActivity.hosPercentage,
      revengePercentage: memberChainActivity.revengePercentage,
      peakHours: memberChainActivity.peakHours,
      sleepPeriod: memberChainActivity.sleepPeriod, // Sleep period
      activeRanges: memberChainActivity.activeRanges, // Active ranges
      activityScore: activityScore
    }
    
    // Calculate overall combat power score
    const combatPower = calculateCombatPowerScore(memberInfo)
    memberInfo.combatPowerScore = combatPower.totalScore
    memberInfo.combatPowerComponents = combatPower.components
    
    memberAnalysis.push(memberInfo)
  })
  
  console.log(`Member analysis complete - processed ${memberAnalysis.length} members`)
  return memberAnalysis.sort((a, b) => b.combatPowerScore - a.combatPowerScore) // Sort by overall combat power score
}

// Calculate faction average sleep period
const calculateFactionAverageSleepPeriod = (validSleepPeriods) => {
  if (validSleepPeriods.length === 0) {
    // If there is no valid data, return the default sleep time (2-10 AM)
    return { start: 2, duration: 8 }
  }
  
  // Calculate the average start time for all valid sleep periods
  const avgStartHour = validSleepPeriods.reduce((sum, period) => sum + period.start, 0) / validSleepPeriods.length
  const avgDuration = validSleepPeriods.reduce((sum, period) => sum + period.duration, 0) / validSleepPeriods.length
  
  console.log(`Faction average sleep period calculation - Valid data points: ${validSleepPeriods.length}, Avg start hour: ${avgStartHour.toFixed(1)}, Avg duration: ${avgDuration.toFixed(1)}`)
  
  return {
    start: Math.round(avgStartHour) % 24,
    duration: Math.round(avgDuration)
  }
}

// Analyze a single member's activity in Chains
const analyzeMemberChainActivity = (memberId, chains, memberName = 'Unknown') => {
  let fourMonthAttacks = 0
  let oneMonthAttacks = 0
  let hosAttacks = 0
  let revengeAttacks = 0
  const timeZoneHours = new Array(24).fill(0)
  const oneMonthAgo = Math.floor(Date.now() / 1000) - (30 * 24 * 3600)
  
  chains.forEach((chainData) => {
    if (chainData.report && chainData.report.attackers) {
      // Find the member in the attackers array
      const memberAttacker = chainData.report.attackers.find(attacker => String(attacker.id) === String(memberId))
      if (memberAttacker && memberAttacker.attacks) {
        const attacks = memberAttacker.attacks
        
        const totalAttacks = attacks.total || 0
        fourMonthAttacks += totalAttacks
        hosAttacks += attacks.hospitalize || 0
        revengeAttacks += attacks.retaliations || 0
        
        // Check if the Chain is within the last month
        if (chainData.report.start >= oneMonthAgo) {
          oneMonthAttacks += totalAttacks
        }
        
        // Active period analysis - improved algorithm
        if (totalAttacks > 0) {
          const chainStart = new Date(chainData.report.start * 1000)
          const chainEnd = new Date(chainData.report.end * 1000)
          const chainDuration = (chainData.report.end - chainData.report.start) / 3600 // hours
          
          // Estimate attack distribution based on Chain duration and number of attacks
          for (let i = 0; i < totalAttacks; i++) {
            // Evenly distribute attack times within the Chain duration
            const attackOffset = (chainDuration * i / totalAttacks) // Relative time of the attack within the Chain (hours)
            const attackTime = new Date(chainStart.getTime() + attackOffset * 3600 * 1000)
            const beijingHour = (attackTime.getUTCHours()) % 24  // Changed from +8 to UTC (no offset)
            timeZoneHours[beijingHour]++
          }
        }
      }
    }
  })
  
  // New sleep period detection algorithm
  const sleepPeriod = findSleepPeriod(timeZoneHours, fourMonthAttacks)
  const activeRanges = calculateActiveRanges(sleepPeriod)
  
  return {
    fourMonthAttacks,
    oneMonthAttacks,
    hosPercentage: fourMonthAttacks > 0 ? (hosAttacks / fourMonthAttacks * 100) : 0,
    revengePercentage: fourMonthAttacks > 0 ? (revengeAttacks / fourMonthAttacks * 100) : 0,
    peakHours: activeRanges.activeHours, // Now an array of active hours
    sleepPeriod: sleepPeriod, // New: sleep period
    activeRanges: activeRanges.ranges, // New: active ranges
    timeZoneDistribution: timeZoneHours
  }
}

// Find sleep period (7-10 hours of continuous low activity)
const findSleepPeriod = (timeZoneHours, totalAttacks) => {
  if (totalAttacks === 0) {
    // When there is no data, assume default sleep time is 2-10 AM
    return { start: 2, duration: 8 }
  }
  
  // Calculate the activity ratio for each hour
  const maxAttacks = Math.max(...timeZoneHours)
  const activityRatios = timeZoneHours.map(count => maxAttacks > 0 ? count / maxAttacks : 0)
  
  let bestSleepPeriod = null
  let lowestAvgActivity = 1.0
  
  // Try sleep periods of 7-10 hours
  for (let duration = 7; duration <= 10; duration++) {
    for (let startHour = 0; startHour < 24; startHour++) {
      let totalActivity = 0
      
      // Calculate the average activity for this period
      for (let i = 0; i < duration; i++) {
        const hour = (startHour + i) % 24
        totalActivity += activityRatios[hour]
      }
      
      const avgActivity = totalActivity / duration
      
      // Find the period with the lowest activity
      if (avgActivity < lowestAvgActivity) {
        lowestAvgActivity = avgActivity
        bestSleepPeriod = { start: startHour, duration: duration }
      }
    }
  }
  
  // If no suitable sleep period is found, use the default value
  if (!bestSleepPeriod) {
    bestSleepPeriod = { start: 2, duration: 8 }
  }
  
  return bestSleepPeriod
}

// Calculate active periods based on sleep time
const calculateActiveRanges = (sleepPeriod) => {
  const activeHours = []
  const ranges = []
  
  // Generate an array of active hours (all hours except sleep time)
  for (let hour = 0; hour < 24; hour++) {
    const sleepStart = sleepPeriod.start
    const sleepEnd = (sleepPeriod.start + sleepPeriod.duration) % 24
    
    let isSleeping = false
    if (sleepStart < sleepEnd) {
      // Sleep time does not cross midnight (e.g., 2-10)
      isSleeping = hour >= sleepStart && hour < sleepEnd
    } else {
      // Sleep time crosses midnight (e.g., 22-6)
      isSleeping = hour >= sleepStart || hour < sleepEnd
    }
    
    if (!isSleeping) {
      activeHours.push(hour)
    }
  }
  
  // Merge active hours into continuous ranges
  if (activeHours.length > 0) {
    let rangeStart = activeHours[0]
    let rangeEnd = activeHours[0]
    
    for (let i = 1; i < activeHours.length; i++) {
      const currentHour = activeHours[i]
      const prevHour = activeHours[i - 1]
      
      if (currentHour === prevHour + 1 || (prevHour === 23 && currentHour === 0)) {
        // Consecutive hours or continuous across midnight
        rangeEnd = currentHour
      } else {
        // Not continuous, save the current range and start a new one
        ranges.push({ start: rangeStart, end: rangeEnd })
        rangeStart = currentHour
        rangeEnd = currentHour
      }
    }
    
    // Add the last range
    ranges.push({ start: rangeStart, end: rangeEnd })
  }
  
  return { activeHours, ranges }
}

// Get the best time for the faction to start a war
const getBestTimesForFaction = (hourlyWinRates, factionNumber) => {
  if (!hourlyWinRates || hourlyWinRates.length === 0) return []
  
  // Select the win rate field based on the faction number
  const winRateField = factionNumber === 1 ? 'faction1WinRate' : 'faction2WinRate'
  
  // Sort by win rate and take the top 5 best times
  const sortedHours = [...hourlyWinRates]
    .sort((a, b) => b[winRateField] - a[winRateField])
    .slice(0, 5)
    .sort((a, b) => a.hour - b.hour) // Re-sort by hour
  
  return sortedHours
}
</script>

<style scoped>
.faction-comparison-card {
  margin-bottom: 20px;
}

.card-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.api-config-card {
  border: 1px solid #e4e7ed;
}

.api-help-text {
  margin-top: 5px;
}

.progress-section {
  margin: 20px 0;
}

.progress-card {
  border: 1px solid #e4e7ed;
}

.progress-item {
  margin-bottom: 15px;
}

.progress-label {
  font-weight: 500;
  margin-bottom: 8px;
  color: #303133;
}

.progress-text {
  margin-top: 5px;
  font-size: 14px;
  color: #606266;
}

.detailed-progress {
  margin-top: 15px;
}

.progress-detail-item {
  margin-bottom: 10px;
}

.progress-detail-label {
  font-size: 14px;
  font-weight: 500;
  margin-bottom: 5px;
  color: #606266;
}

.progress-detail-text {
  margin-top: 3px;
  font-size: 12px;
  color: #909399;
}

.cache-info-card {
  border: 1px solid #e4e7ed;
}

.cache-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.result-card {
  border: 1px solid #e4e7ed;
}

.result-summary {
  font-size: 14px;
  line-height: 1.6;
}

.result-summary ul {
  margin: 10px 0;
  padding-left: 20px;
}

.result-summary li {
  margin: 5px 0;
}

.win-rate-card {
  border: 1px solid #e4e7ed;
}

.win-rate-display {
  padding: 20px;
}

.faction-win-rate {
  text-align: center;
}

.win-percentage {
  font-size: 24px;
  font-weight: 500;
  margin-bottom: 10px;
}

.analysis-text {
  margin-top: 20px;
}

.strength-analysis-card {
  border: 1px solid #e4e7ed;
}

.overall-comparison {
  margin-bottom: 20px;
}

.member-analysis {
  margin-top: 20px;
}

.member-count-info {
  margin-bottom: 10px;
  font-size: 14px;
  color: #606266;
}

.peak-hours {
  color: #67c23a;
}

.faction-analysis {
  padding: 10px;
}

.faction-analysis h5 {
  margin-bottom: 10px;
  color: #409eff;
  font-weight: 600;
}

.faction-analysis ul {
  list-style: none;
  padding: 0;
  margin: 0;
}

.faction-analysis li {
  padding: 5px 0;
  border-bottom: 1px solid #f0f0f0;
}

.faction-analysis li:last-child {
  border-bottom: none;
}

.analysis-details {
  padding: 10px 0;
}

.cache-data-content {
  max-height: 600px;
  overflow-y: auto;
}

.json-content {
  background: #f5f5f5;
  padding: 15px;
  border-radius: 4px;
  font-family: 'Courier New', monospace;
  font-size: 12px;
  line-height: 1.4;
  white-space: pre-wrap;
  word-wrap: break-word;
  max-height: 500px;
  overflow-y: auto;
  border: 1px solid #e4e7ed;
}

.sleep-period {
  color: #f56c6c;
  font-weight: 500;
}
</style> 