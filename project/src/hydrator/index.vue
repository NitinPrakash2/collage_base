<template>
    <Collage :_p="_p" :__p="_$p" :mode="mode" />
</template>
<script lang="ts" setup>
import type { _p_TYP, _$p_TYP } from "../shared/types";
import { ref } from "vue";
import Collage from "./src/components/collage.vue";
const {_p,_$p} = defineProps<{
    _p:_p_TYP,
    _$p:_$p_TYP,
}>();
const mode = ref<"sticky" | "normal">("normal");

(async () => {
    //==test==//  [START]
    /* await _p.f.wait_until(() => window.location !== null, 50); */

    //1
    _p.my[`emitter`] = _p.f.new_emitter();
    _p.my[`emitter`].on("msg", async (_$:any) => {
        console.log(`_p.my.emitter.on`, _$);
    });
    await _p.my[`emitter`].emit("msg", {
        type: `on:change`,
        _p: _p,
        _$p: _$p,
    });

    
    //2
    _p.f.listen("msg", async (_$) => {
        console.log(`_p.f.listen`, _$);
        if(_$.type==="on:change"){
            const m  = _$.custom?.mode;
            const theme=_$.custom.theme;
            console.log(`Theme changed to: `,theme);
            mode.value=m
             const htmlEl = document.documentElement;
      if (theme === 'dark') {
        htmlEl.classList.add('dark');
      } else {
        htmlEl.classList.remove('dark');
      }
        }
    });
    setTimeout(async () => {
        await _p.f.call("msg", {
            type: `on:change`,
            _p: _p,
            _$p: _$p,
        });
    }, 500);

    //==test==//  [END]
})();

</script>
