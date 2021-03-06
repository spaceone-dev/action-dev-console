<template>
    <p-widget-layout title="Service Accounts">
        <div class="chart-container">
            <p-chart-loader :loading="loading" class="chart">
                <template #loader>
                    <canvas ref="loaderRef" />
                </template>
                <canvas ref="chartRef" />
            </p-chart-loader>
        </div>
        <div class="legends">
            <template v-if="loading">
                <div v-for="v in skeletons" :key="v" class="flex items-center p-4">
                    <p-skeleton width="1.5rem" height="1.5rem" class="mr-4 flex-shrink-0" />
                    <p-skeleton class="flex-grow" />
                </div>
            </template>
            <p-grid-layout v-else :items="data" row-gap="0.5rem"
                           column-gap="0" :fix-column="1" card-min-width="0"
                           card-height="auto" :card-class="() => []"
            >
                <template #card="{item, index}">
                    <router-link :to="item.href">
                        <p-selectable-item :icon-url="item.icon" theme="card"
                                           default-icon="ic_provider_other"
                        >
                            <template #contents>
                                <div v-tooltip.bottom="{content: item.name, delay: {show: 500}}"
                                     class="mx-2 text-base truncate leading-tight"
                                >
                                    {{ item.name }}
                                </div>
                            </template>
                            <template #extra>
                                <p-badge :background-color="item.color" class="count">
                                    {{ item.count }}
                                </p-badge>
                            </template>
                        </p-selectable-item>
                    </router-link>
                </template>
            </p-grid-layout>
        </div>
    </p-widget-layout>
</template>

<script lang="ts">
import {
    getCurrentInstance, toRefs, watch,
} from '@vue/composition-api';
import PWidgetLayout from '@/components/organisms/layouts/widget-layout/WidgetLayout.vue';
import PBadge from '@/components/atoms/badges/Badge.vue';
import PGridLayout from '@/components/molecules/layouts/grid-layout/GridLayout.vue';
import PSelectableItem from '@/components/molecules/selectable-item/SelectableItem.vue';
import PSkeleton from '@/components/atoms/skeletons/Skeleton.vue';
import PChartLoader from '@/components/organisms/charts/chart-loader/ChartLoader.vue';
import { SPieChart } from '@/lib/chart/pie-chart';
import { ProviderInfo, ProviderStoreType, useStore } from '@/store/toolset';
import { violet, yellow } from '@/styles/colors';
import _ from 'lodash';
import Color from 'color';
import { fluentApi } from '@/lib/fluent-api';
import { SChartToolSet } from '@/lib/chart/toolset';
import { STAT_OPERATORS } from '@/lib/fluent-api/statistics/type';
import {
    serviceAccountsProps,
    ServiceAccountsPropsType,
} from '@/views/common/widgets/service-accounts/ServiceAccounts.toolset';

export default {
    name: 'ServiceAccounts',
    components: {
        PWidgetLayout,
        PBadge,
        PGridLayout,
        PSelectableItem,
        PSkeleton,
        PChartLoader,
    },
    props: serviceAccountsProps,
    setup(props: ServiceAccountsPropsType) {
        const vm: any = getCurrentInstance();

        const {
            provider,
        } = useStore();
        const providerStore: ProviderStoreType = provider;

        interface Value {
            provider: string;
            count: number;
        }

        const api = fluentApi.statisticsTest().resource().stat<Value>()
            .addGroupKey('provider', 'provider')
            .addGroupField('count', STAT_OPERATORS.count)
            .setSort('count');

        interface Item {
            provider: string;
            name: string;
            icon: string;
            color: string;
            count: number;
            href: string;
        }
        interface StateInterface {
            loaderRef: HTMLCanvasElement | null;
            data: Item[];
            loading: boolean;
        }

        const ts = new SChartToolSet<SPieChart, StateInterface>(SPieChart,
            chart => chart.addData(_.map(ts.state.data, d => d.count), 'Account')
                .setLabels(_.map(ts.state.data, d => d.name))
                .setColors(_.map(ts.state.data, d => d.color))
                .setDefaultCount(4)
                .apply(), {
                loaderRef: null,
                data: [],
                loading: true,
            });

        watch(() => ts.state.loaderRef, () => {
            if (!ts.state.loaderRef) return;
            new SPieChart(ts.state.loaderRef)
                .addData([8, 2], '').setTooltipEnabled(false)
                .setColors([violet[200], Color(violet[200]).alpha(0.5).toString()])
                .setBorderWidth(0)
                .setShowTotalCount(false)
                .setAnimationDuration(0)
                .apply();
        });

        const getData = async (): Promise<void> => {
            ts.state.loading = true;
            ts.state.data = [];
            await providerStore.getProvider();
            try {
                const res = await props.getAction(api).execute();
                const others: Item = {
                    name: 'Others',
                    icon: 'ic_provider_other',
                    color: yellow[500],
                    count: 0,
                    provider: '',
                    href: '/identity/service-account',
                };
                const providers: ProviderInfo = providerStore.state.providers;

                if (res.data.results.length > 0) {
                    _.forEach(res.data.results, (d: Value, i) => {
                        if (providers[d.provider]) {
                            ts.state.data.push({
                                ...providers[d.provider],
                                href: `/identity/service-account?p=1&ps=15&provider=${d.provider}`,
                                count: d.count,
                            });
                        } else others.count += d.count;
                    });
                } else {
                    ts.state.data = _.map(providers, p => ({ ...p, count: 0 }));
                }
                ts.state.data.push(others);
            } catch (e) {
                console.error(e);
            } finally {
                ts.state.loading = false;
            }
        };

        getData();

        return {
            ...toRefs(ts.state),
            skeletons: _.range(4),
            onSelected(item): void {
                vm.$router.push('/identity/service-account');
            },
        };
    },
};
</script>

<style lang="postcss" scoped>
    .chart {
        height: 11.25rem;
    }
    .count {
        font-size: 0.875rem;
        font-weight: bold;
    }
    .legends {
        @apply w-full flex-grow justify-center items-center m-auto;
    }
    .chart-container {
        @apply flex justify-center items-center mb-4;
    }
</style>
