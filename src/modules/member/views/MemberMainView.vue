<template>
    <div class="subject-card-wrapper">
        <subject-card-comp></subject-card-comp>
        <my-opinion-card-comp></my-opinion-card-comp>
    </div>
</template>

<script>
import SubjectCardComp from '@/modules/member/components/SubjectCardComp.vue';
import MyOpinionCardComp from '@/modules/member/components/MyOpinionCardComp.vue';

export default {
    name: 'MemberMainView',
    components: {
        SubjectCardComp, 
        MyOpinionCardComp
    }
}

</script>

<style scoped>
/* 스크롤바 관련 설정 */
  .subject-card-wrapper {
  height: 600px;
  overflow-y: auto;
  /* margin-top: 60px;
  margin-bottom: 60px; */
}

/* 스크롤바 숨기기 */
.subject-card-wrapper::-webkit-scrollbar {
  display: none;
}

</style>