<template>
  <div class="flex min-h-screen bg-gradient-to-b from-gray-900 to-black text-gray-200">
    <Sidebar />

    <div class="flex-1 flex flex-col">
      <DashboardHeader title="Gestion des Catégories" />

      <main class="p-6 space-y-6">
        <div v-if="loading" class="text-center py-6">
          <p class="text-gray-400">Chargement...</p>
        </div>

        <div v-else>
          <div class="flex justify-between items-center mb-6">
            <h2 class="text-xl font-semibold text-gray-300">Catégories</h2>
            <button
                @click="openModal('create')"
                class="bg-blue-600 hover:bg-blue-500 text-white px-4 py-2 rounded-lg"
            >
              Ajouter une catégorie
            </button>
          </div>

          <div class="bg-gray-800 p-4 rounded-lg">
            <ul class="space-y-4">
              <CategoryTree
                  v-for="category in categories"
                  :key="category.id"
                  :category="category"
                  @edit="openModal('edit', $event)"
                  @delete="deleteCategoryHandler"
              />
            </ul>
          </div>
        </div>

        <Modal
            v-if="modalVisible"
            :visible="modalVisible"
            :title="modalMode === 'create' ? 'Ajouter une catégorie' : 'Modifier une catégorie'"
            @close="closeModal"
        >
          <form @submit.prevent="modalMode === 'create' ? createCategoryHandler() : updateCategoryHandler()">
            <div class="space-y-4">
              <label class="block">
                <span class="text-gray-300">Nom de la catégorie</span>
                <input
                    v-model="categoryForm.name"
                    type="text"
                    class="mt-1 block w-full bg-gray-800 text-white border border-gray-700 rounded-lg px-3 py-2"
                    required
                />
              </label>
              <label class="block">
                <span class="text-gray-300">Description</span>
                <textarea
                    v-model="categoryForm.description"
                    class="mt-1 block w-full bg-gray-800 text-white border border-gray-700 rounded-lg px-3 py-2"
                    rows="4"
                    required
                ></textarea>
              </label>
              <label class="block">
                <span class="text-gray-300">Parent</span>
                <select
                    v-model="categoryForm.parent_id"
                    class="mt-1 block w-full bg-gray-800 text-white border border-gray-700 rounded-lg px-3 py-2"
                >
                  <option value="">Aucune</option>
                  <option v-for="parent in categories" :key="parent.id" :value="parent.id">
                    {{ parent.name }}
                  </option>
                </select>
              </label>
            </div>
            <div class="mt-6 flex justify-end space-x-4">
              <button
                  type="button"
                  @click="closeModal"
                  class="bg-gray-600 hover:bg-gray-500 text-white px-4 py-2 rounded-lg"
              >
                Annuler
              </button>
              <button
                  type="submit"
                  class="bg-blue-600 hover:bg-blue-500 text-white px-4 py-2 rounded-lg"
              >
                Enregistrer
              </button>
            </div>
          </form>
        </Modal>
      </main>
    </div>
  </div>
</template>

<script lang="ts" setup>
import { ref, onMounted } from "vue";
import Sidebar from "../components/Dashboard/Sidebar.vue";
import DashboardHeader from "../components/Dashboard/DashboardHeader.vue";
import Modal from "../components/Dashboard/Modal.vue";
import CategoryTree from "../components/Dashboard/CategoryTree.vue";
import { useToast } from "../composables/Toast";
import { getCategories, createCategory, updateCategory, deleteCategory } from "../api/admin";

const { success, error } = useToast();

const categories = ref([]);
const loading = ref(false);

const modalVisible = ref(false);
const modalMode = ref("create");
const categoryForm = ref({ id: null, name: "", description: "", parent_id: null });

const fetchCategories = async () => {
  try {
    loading.value = true;
    const response = await getCategories();
    categories.value = organizeCategories(response.data);
  } catch (err) {
    error("Erreur lors du chargement des catégories.");
    console.error(err);
  } finally {
    loading.value = false;
  }
};

const organizeCategories = (categories) => {
  const categoryMap = new Map();

  categories.forEach((category) => {
    category.children = [];
    categoryMap.set(category.id, category);
  });

  const organizedCategories = [];
  categories.forEach((category) => {
    if (category.parent_id) {
      const parent = categoryMap.get(category.parent_id);
      if (parent) parent.children.push(category);
    } else {
      organizedCategories.push(category);
    }
  });

  return organizedCategories;
};

const createCategoryHandler = async () => {
  if (!categoryForm.value.name.trim() || !categoryForm.value.description.trim()) {
    error("Le nom et la description de la catégorie sont requis.");
    return;
  }

  try {
    await createCategory({
      name: categoryForm.value.name,
      description: categoryForm.value.description,
      parent_id: categoryForm.value.parent_id || null,
    });
    success("Catégorie ajoutée avec succès !");
    fetchCategories();
    closeModal();
  } catch (err) {
    error("Erreur lors de la création de la catégorie.");
    console.error(err);
  }
};

const updateCategoryHandler = async () => {
  if (!categoryForm.value.name.trim() || !categoryForm.value.description.trim()) {
    error("Le nom et la description de la catégorie sont requis.");
    return;
  }

  try {
    await updateCategory({
      id: categoryForm.value.id,
      name: categoryForm.value.name,
      description: categoryForm.value.description,
      parent_id: categoryForm.value.parent_id || null,
    });
    success("Catégorie mise à jour avec succès !");
    fetchCategories();
    closeModal();
  } catch (err) {
    error("Erreur lors de la mise à jour de la catégorie.");
    console.error(err);
  }
};

const deleteCategoryHandler = async (id) => {
  if (confirm("Êtes-vous sûr de vouloir supprimer cette catégorie ?")) {
    try {
      await deleteCategory(id);
      success("Catégorie supprimée avec succès !");
      fetchCategories();
    } catch (err) {
      error("Erreur lors de la suppression de la catégorie.");
      console.error(err);
    }
  }
};

const openModal = (mode, category = null) => {
  modalMode.value = mode;
  modalVisible.value = true;
  categoryForm.value = category
      ? {
        id: category.id,
        name: category.name,
        description: category.description,
        parent_id: category.parent_id || null,
      }
      : { id: null, name: "", description: "", parent_id: null };
};

const closeModal = () => {
  modalVisible.value = false;
  categoryForm.value = { id: null, name: "", description: "", parent_id: null };
};

onMounted(() => fetchCategories());
</script>
