<template>
  <div style="border-bottom: solid gray">
    <h2 style="text-align: center">Quản lý tiêu chí đánh giá</h2>
  </div>
  <div class="content">
    <nav class="navbar navbar-light mt-3">
      <select
        class="p-2 pe-3 rounded-3"
        aria-label="Default select example"
        v-model="selectedDepartmentId"
        @change="handleDepartmentChange"
      >
        <option
          v-for="department in departments"
          :key="department.id"
          :value="department.id"
        >
          {{ department.name }}
        </option>
      </select>
      <input
        type="text"
        v-model="searchQuery"
        placeholder="Tìm kiếm tiêu chí đánh giá..."
        class="search-bar"
      />
      <button
        class="btn btn-success me-3"
        type="button"
        @click="openAddCriteriaModal"
      >
        Thêm tiêu chí
      </button>
    </nav>
    <div class="table-responsive-md mt-2">
      <table class="table table-hover table-bordered criteria-table">
        <thead>
          <tr>
            <th scope="col">STT</th>
            <th scope="col">Tên tiêu chí đánh giá</th>
            <th scope="col">Loại tiêu chí đánh giá</th>
            <th scope="col">Số điểm</th>
            <th scope="col">Tác vụ</th>
          </tr>
        </thead>
        <tbody>
          <tr v-for="(criteria, index) in paginatedCriterias" :key="criteria.id">
            <td>{{ index + 1 }}</td>
            <td class="text-start">{{ criteria.title }}</td>
            <td class="text-start">{{ criteria.visibleFor }}</td>
            <td>{{ criteria.point }}</td>
            <td>
              <button class="btn btn-primary me-3">
                <router-link
                  :to="`/detail-criterias/${criteria.id}`"
                  class="nav-link"
                  active-class="active"
                >Chi tiết</router-link>
              </button>

              <a
                type="button"
                class="btn btn-warning me-3"
                @click="editCriterias(criteria)"
                >Sửa</a
              >
              <button
                type="button"
                class="btn btn-danger"
                @click="confirmDeleteCriterias(criteria.id)"
              >
                Xoá
              </button>
            </td>
          </tr>
        </tbody>
      </table>
    </div>

    <!-- Pagination -->
    <div class="pagination-wrapper">
      <button
        @click="prevPage"
        :disabled="currentPage === 1"
        class="pagination-btn"
      >
        <i class="fas fa-arrow-left"></i>
      </button>
      <span>Trang {{ currentPage }} / {{ totalPages }}</span>
      <button
        @click="nextPage"
        :disabled="currentPage === totalPages"
        class="pagination-btn"
      >
        <i class="fas fa-arrow-right"></i>
      </button>
    </div>

    <!-- Modal Component -->
    <AddCriteriasModal
      :is-visible="isAddCriteriaModalVisible"
      @close="closeAddCriteriaModal"
      @criteria-added="fetchCriterias"
    />
    <EditCriteriasModal
      :is-visible1="isModalVisible1"
      :criteriasData="selectedCriterias"
      @close="closeCriteriasEditModal"
      @criterias-edited="fetchCriterias"
    />
  </div>
</template>

<script>
import EditCriteriasModal from "./modal/Criterias/EditCriteriasModal.vue";
import Swal from "sweetalert2";
import CriteriasService from "@/services/CriteriasService";
import AddCriteriasModal from "./modal/Criterias/AddCriteriasModal.vue";
import { toast } from "vue3-toastify";
import "vue3-toastify/dist/index.css";

export default {
  components: {
    EditCriteriasModal,
    AddCriteriasModal,
  },
  data() {
    return {
      isModalVisible1: false,
      isAddCriteriaModalVisible: false,
      selectedCriterias: null,
      criterias: [],
      departments: [], // Danh sách phòng ban
      selectedDepartmentId: 0, // ID của phòng ban được chọn
      selectedDepartment: null, // Object của phòng ban được chọn
      currentPage: 1,
      itemsPerPage: 10,
      searchQuery: "", // Dùng cho tìm kiếm tiêu chí
    };
  },
  mounted() {
    this.fetchDepartments(); // Lấy danh sách phòng ban khi component được mount
  },
  computed: {
    totalPages() {
      return Math.ceil(this.criterias.length / this.itemsPerPage);
    },

    paginatedCriterias() {
      const start = (this.currentPage - 1) * this.itemsPerPage;
      const end = start + this.itemsPerPage;
      return this.criterias.slice(start, end);
    },
  },
  methods: {
    // Lấy danh sách phòng ban từ API
    async fetchDepartments() {
      try {
        const response = await CriteriasService.fetchDepartment();
        if (response.code === 1010) {
          this.departments = response.data;

          if (this.departments.length > 0) {
            this.selectedDepartmentId = this.departments[0].id;
            
            this.handleDepartmentChange();
          }
        }
      } catch (error) {
        console.error("Error fetching departments:", error);
      }
    },

    // Cập nhật tiêu chí dựa trên phòng ban được chọn
    handleDepartmentChange() {
      this.selectedDepartment = this.departments.find(
        (dept) => dept.id === this.selectedDepartmentId
      );

      if (this.selectedDepartment) {
        this.criterias = this.selectedDepartment.criteria.filter(
          (criteria) => !criteria.deleted,
          localStorage.setItem("department_id",this.selectedDepartmentId)
        );
      } else {
        this.criterias = [];
      }
    },

    editCriterias(criteria) {
      this.selectedCriterias = { ...criteria };
      this.isModalVisible1 = true;
    },

    async confirmDeleteCriterias(id) {
      const result = await Swal.fire({
        title: "Bạn có muốn xóa tiêu chí đánh giá này?",
        icon: "warning",
        showCancelButton: true,
      });
      if (result.isConfirmed) {
        try {
          const res = await CriteriasService.deletedCriterias(id);
          console.log("res: ", res)
          if (res.status === 204) {
            toast.success("Tiêu chí đã được xóa thành công.", {
              autoClose: 2000,
            });
            this.handleDepartmentChange();
          }
        } catch (error) {
          console.error("Lỗi khi xóa tiêu chí:", error);
          Swal.fire({
            title: "Lỗi!",
            text: "Đã xảy ra lỗi khi xóa tiêu chí.",
            icon: "error",
          });
        }
      }
    },

    openAddCriteriaModal() {
      this.isAddCriteriaModalVisible = true;
    },

    closeAddCriteriaModal() {
      this.isAddCriteriaModalVisible = false;
    },

    closeCriteriasEditModal() {
      this.isModalVisible1 = false;
    },

    prevPage() {
      if (this.currentPage > 1) {
        this.currentPage--;
      }
    },
    nextPage() {
      if (this.currentPage < this.totalPages) {
        this.currentPage++;
      }
    },
    goToPage(page) {
      this.currentPage = page;
    },
  },
};
</script>

<style scoped>
.dropdown-toggle {
  color: #007bff;
  font-weight: bold;
  text-decoration: none;
  padding: 0.5em 1em;
  cursor: pointer;
}

.dropdown:hover .dropdown-menu {
  display: block;
  margin-top: 0;
}

.dropdown-toggle::after {
  display: none;
}

.search-bar {
  padding: 10px;
  width: 350px;
  border-radius: 25px;
  border: 1px solid #ddd;
  outline: none;
  box-shadow: 0px 2px 4px rgba(0, 0, 0, 0.1);
  transition: border-color 0.3s ease;
}

.search-bar:focus {
  border-color: #007bff;
}

.criteria-table {
  border-collapse: separate;
  border-spacing: 0;
  width: 100%;
  max-height: 1000vh;
  text-align: left;
  background-color: white;
  border-radius: 8px;
  overflow: hidden;
  box-shadow: 0px 4px 10px rgba(0, 0, 0, 0.05);
  position: relative;
}

.criteria-table th,
.criteria-table td {
  padding: 7px;
  vertical-align: middle;
  border-bottom: 1px solid #f0f0f0;
  text-align: center;
}

.criteria-table th {
  background-color: #000066;
  color: white;
  font-weight: 600;
  font-size: 20px;
  cursor: pointer;
}

.criteria-table tr:hover {
  background-color: #f9f9f9;
}

.criteria-table tr:nth-child(even) {
  background-color: #f2f2f2;
}

.pagination-wrapper {
  display: flex;
  justify-content: center;
  margin-top: 30px;
}

.pagination-wrapper span {
  margin-top: 28px;
}

.pagination-btn {
  margin: 10px 10px;
  padding: 8px 16px;
  background-color: #007bff;
  color: white;
  border: none;
  border-radius: 5px;
  cursor: pointer;
  transition: background-color 0.3s ease;
  box-shadow: 0px 2px 4px rgba(0, 0, 0, 0.1);
  margin-top: 20px;
}

.pagination-btn:hover {
  background-color: #0056b3;
}

.pagination-btn:disabled {
  background-color: #aaa;
  cursor: not-allowed;
}

.modal.show {
  display: block;
  background-color: rgba(0, 0, 0, 0.5);
}

.modal-body input {
  background-color: #fff;
  width: 100%;
}
</style>
