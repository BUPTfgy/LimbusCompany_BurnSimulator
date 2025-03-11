<template>
  <div class="app-container">
    <div class="left-panel">
      <h2>技能文件</h2>
      <input type="file" @change="loadSkillsFile" accept=".json" class="file-input" />
      <textarea v-model="skillsJson" class="json-editor"></textarea>
      <button @click="downloadSkills" class="btn btn-primary">下载技能文件</button>
    </div>

    <div class="right-panel">
      <h2>目标设置</h2>
      <div class="target-settings">
        <div class="form-group">
          <label>初始烧伤强度:</label>
          <input type="number" v-model.number="targetBurnDamage" class="form-control" />
        </div>
        <div class="form-group">
          <label>初始烧伤层数:</label>
          <input type="number" v-model.number="targetBurnStacks" class="form-control" />
        </div>
      </div>

      <h2>技能列表</h2>
      <div class="skill-list">
        <draggable v-model="selectedSkills" @end="onDragEnd" :options="{ animation: 200 }">
          <template #item="{ element }">
            <div class="skill-item">
              {{ element.skill_name }}
              <button @click="removeSkill(selectedSkills.indexOf(element))" class="btn btn-danger btn-sm">移除</button>
            </div>
          </template>
        </draggable>
        <p v-if="selectedSkills.length === 0" class="empty-list">
          当前技能列表为空。
        </p>
      </div>

      <select v-model="selectedSkill" @change="addSkill" class="form-control skill-select">
        <option value="" disabled>选择技能</option>
        <option v-for="skill in skills" :key="skill.skill_name" :value="skill">
          {{ skill.skill_name }}
        </option>
      </select>

      <h2>烧伤模拟结果</h2>
      <div class="result-table">
        <table class="table table-bordered">
          <thead>
            <tr>
              <th>技能</th>
              <th>烧伤强度</th>
              <th>烧伤层数</th>
            </tr>
          </thead>
          <tbody>
            <tr>
              <td>初始状态</td>
              <td>{{ burnResults[0].damage }}</td>
              <td>{{ burnResults[0].stacks }}</td>
            </tr>
            <tr v-for="(result, index) in burnResults.slice(1)" :key="index">
              <td>{{ selectedSkills[index]?.skill_name || 'N/A' }}</td>
              <td>{{ result.damage }}</td>
              <td>{{ result.stacks }}</td>
            </tr>
          </tbody>
        </table>
      </div>
    </div>
  </div>
</template>

<script>
import draggable from 'vuedraggable';

export default {
  components: {
    draggable
  },
  data() {
    return {
      skillsJson: '[]',
      skills: [],
      selectedSkill: '',
      selectedSkills: [],
      targetBurnDamage: 0,
      targetBurnStacks: 0,
      burnResults: [{ damage: 0, stacks: 0 }]
    };
  },
  watch: {
    targetBurnDamage(newVal) {
      this.burnResults[0].damage = newVal;
      this.recalculateBurn();
    },
    targetBurnStacks(newVal) {
      this.burnResults[0].stacks = newVal;
      this.recalculateBurn();
    }
  },
  mounted() {
    // 尝试加载默认的 skills.json 文件
    fetch('skills.json')
      .then(response => response.json())
      .then(data => {
        this.skills = data;
        this.skillsJson = JSON.stringify(data, null, 2); // 格式化 JSON
      })
      .catch(error => {
        console.warn('Failed to load default skills.json:', error);
      });
  },
  methods: {
    loadSkillsFile(event) {
      const file = event.target.files[0];
      if (file) {
        const reader = new FileReader();
        reader.onload = (e) => {
          try {
            this.skillsJson = e.target.result;
            this.skills = JSON.parse(this.skillsJson);
          } catch (error) {
            alert('JSON 文件解析错误！');
          }
        };
        reader.readAsText(file);
      }
    },
    downloadSkills() {
      try {
        const data = JSON.parse(this.skillsJson);
        const json = JSON.stringify(data, null, 2);
        const blob = new Blob([json], { type: 'application/json' });
        const url = URL.createObjectURL(blob);
        const a = document.createElement('a');
        a.href = url;
        a.download = 'skills.json';
        document.body.appendChild(a);
        a.click();
        document.body.removeChild(a);
        URL.revokeObjectURL(url);
      } catch (error) {
        alert('JSON 文件格式不正确！');
      }
    },
    addSkill() {
      if (this.selectedSkill) {
        this.selectedSkills.push(this.selectedSkill);
        this.selectedSkill = '';
        this.recalculateBurn();
      }
    },
    removeSkill(index) {
      this.selectedSkills.splice(index, 1);
      this.recalculateBurn();
    },
    onDragEnd() {
      this.recalculateBurn();
    },
    recalculateBurn() {
      this.burnResults = [{ damage: this.targetBurnDamage, stacks: this.targetBurnStacks }];
      let currentDamage = this.targetBurnDamage;
      let currentStacks = this.targetBurnStacks;

      for (const skill of this.selectedSkills) {
        let newDamage = currentDamage;
        let newStacks = currentStacks;

        // 应用技能的烧伤效果 (考虑所有硬币)
        const coins = [
          { damage: skill["硬币1烧伤强度"], stacks: skill["硬币1烧伤层数"] },
          { damage: skill["硬币2烧伤强度"], stacks: skill["硬币2烧伤层数"] },
          { damage: skill["硬币3烧伤强度"], stacks: skill["硬币3烧伤层数"] },
          { damage: skill["硬币4烧伤强度"], stacks: skill["硬币4烧伤层数"] }
        ];

        for (const coin of coins) {
          if (coin.stacks === 0 && coin.damage > 0 && currentStacks ===0 ) {
            newStacks += 1;
          }
          if (coin.stacks > 0 ) {
            //烧伤强度不能增加烧伤层数
            newStacks += coin.stacks;
          }
          newDamage += coin.damage;
        }

        this.burnResults.push({ damage: newDamage, stacks: newStacks });
        currentDamage = newDamage;
        currentStacks = newStacks;
      }
    }
  }
};
</script>

<style scoped>
/* General Styles */
body {
  font-family: 'Arial', sans-serif;
  background-color: #f8f9fa;
}

.app-container {
  display: flex;
  height: 100vh;
  padding: 20px;
}

h2 {
  margin-top: 0;
  margin-bottom: 20px;
  color: #343a40;
}

/* Left Panel Styles */
.left-panel {
  width: 400px;
  padding: 20px;
  border-right: 1px solid #dee2e6;
  background-color: #fff;
}

.file-input {
  margin-bottom: 10px;
}

.json-editor {
  width: 100%;
  height: 300px;
  margin-bottom: 10px;
  border: 1px solid #ced4da;
  padding: 10px;
  font-family: monospace;
  resize: vertical;
}

/* Right Panel Styles */
.right-panel {
  flex: 1;
  padding: 20px;
  background-color: #fff;
}

/* Target Settings Styles */
.target-settings {
  display: flex;
  gap: 20px;
  margin-bottom: 20px;
}

.form-group {
  display: flex;
  flex-direction: column;
}

label {
  margin-bottom: 5px;
  color: #495057;
}

.form-control {
  padding: 8px 12px;
  border: 1px solid #ced4da;
  border-radius: 4px;
  font-size: 1rem;
  color: #495057;
}

/* Skill List Styles */
.skill-list {
  min-height: 50px;
  margin-bottom: 20px;
  border: 1px dashed #ced4da;
  padding: 10px;
}

.skill-item {
  background-color: #f0f0f0;
  padding: 10px;
  margin: 5px 0;
  border-radius: 4px;
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.empty-list {
  font-style: italic;
  color: #6c757d;
}

/* Skill Select Styles */
.skill-select {
  width: 100%;
  padding: 10px;
  border-radius: 4px;
  border: 1px solid #ced4da;
}

/* Result Table Styles */
.result-table {
  margin-top: 20px;
}

.table {
  width: 100%;
  border-collapse: collapse;
}

.table th,
.table td {
  border: 1px solid #dee2e6;
  padding: 8px;
  text-align: left;
}

.table th {
  background-color: #f2f2f2;
}

/* Button Styles */
.btn {
  padding: 8px 12px;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  color: white;
}

.btn-primary {
  background-color: #007bff;
}

.btn-danger {
  background-color: #dc3545;
}

/* Utility Styles */
.mt-2 {
  margin-top: 0.5rem;
}

.mb-2 {
  margin-bottom: 0.5rem;
}
</style>