<template>
  <div class="subject-card">
    <admin-subject-comp></admin-subject-comp>
  </div>
</template>

<script>
import AdminSubjectComp from '@/modules/admin/components/AdminSubjectComp.vue';

export default {
  name: 'AdminSubjectView',
  components: {
    AdminSubjectComp
  },
};
</script>

<!-- <style>
/* 스크롤바 관련 설정 */
 .subject-card {
  max-height: 1500px;
  overflow-y: auto;
  margin-top: 60px;
  margin-bottom: 60px;
} 

/* 스크롤바 숨기기 */
 .subject-card ::-webkit-scrollbar {
  display: none;
} 
</style> -->