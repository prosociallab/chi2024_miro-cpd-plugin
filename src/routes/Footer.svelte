<div class="tabbar">
  <TabBar {tabs} let:tab bind:active>
    <Tab
      {tab}
      stacked={true}
      indicatorSpanOnlyContent={true}
      tabIndicator$transition="fade"
      href={tab.url}
      style="color: transparent !important"
    >
      <Icon class="material-icons">{tab.icon}</Icon>
      <Label>{tab.label}</Label>
    </Tab>
  </TabBar>
</div>

<script lang="ts">
  import Tab, { Icon, Label } from '@smui/tab';
  import TabBar from '@smui/tab-bar';
  import { page } from '$app/stores';

  let tabs = [
    {
      k: 1,
      icon: 'create',
      label: '정보 입력',
      url: "/"
    },
    {
      k: 2,
      icon: 'equalizer',
      label: '결과',
      url: "result"
    },
    {
      k: 3,
      icon: 'info',
      label: '상세',
      url: "detail"
    },
    {
      k: 4,
      icon: 'code',
      label: 'Code',
      url: "code"
    },
  ];
  let active = $page.route.id == "/" ? tabs[0] : ($page.route.id == "/result" ? tabs[1]:$page.route.id == "/detail" ? tabs[2]:tabs[3]);
</script>

<style>
  .tabbar {
    border-top: 1px solid rgb(208, 208, 208);
    position: fixed;
    left: 0;
    bottom: 0;
    width: 100%;
    color: white;
    background-color: white;
    text-align: center;
  }
</style>