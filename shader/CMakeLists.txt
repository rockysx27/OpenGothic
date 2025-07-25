cmake_minimum_required(VERSION 3.12)

project(GothicShaders)

set(HEADER "${PROJECT_BINARY_DIR}/sprv/shader.h")
set(CPP    "${PROJECT_BINARY_DIR}/sprv/shader.cpp")

file(GLOB_RECURSE SOURCES
  "${CMAKE_SOURCE_DIR}/shader/*.vert"
  "${CMAKE_SOURCE_DIR}/shader/*.tesc"
  "${CMAKE_SOURCE_DIR}/shader/*.tese"
  "${CMAKE_SOURCE_DIR}/shader/*.geom"
  "${CMAKE_SOURCE_DIR}/shader/*.frag"
  "${CMAKE_SOURCE_DIR}/shader/*.glsl"
  "${CMAKE_SOURCE_DIR}/shader/*.task"
  "${CMAKE_SOURCE_DIR}/shader/*.mesh"
  "${CMAKE_SOURCE_DIR}/shader/*.comp")

file(GLOB_RECURSE GLSL_SOURCES
  "${CMAKE_SOURCE_DIR}/shader/*.glsl")

# GLSL to SPIRV compiler
find_program(GLSLANGVALIDATOR glslangValidator "/opt/homebrew/bin")
if(NOT GLSLANGVALIDATOR)
  message(FATAL_ERROR "glslangValidator required")
endif()

function(add_shader OUTPUT SOURCE)
  set(OUTPUT_FILE "${PROJECT_BINARY_DIR}/sprv/${OUTPUT}.sprv")
  set(SOURCE_FILE "${CMAKE_SOURCE_DIR}/shader/${SOURCE}")
  set(OPENGOTHIC_SHADERS      ${OPENGOTHIC_SHADERS}      ${SOURCE_FILE} PARENT_SCOPE)
  set(OPENGOTHIC_SHADERS_SPRV ${OPENGOTHIC_SHADERS_SPRV} ${OUTPUT_FILE} PARENT_SCOPE)

  list(LENGTH ARGV ARGC)
  if(ARGC GREATER 2)
    list(SUBLIST ARGV 2 -1 VARS)
  endif()

  if("${OUTPUT}" MATCHES "\.(mesh)")
    list(APPEND VARS "--target-env" "spirv1.4")
  endif()

  if("${OUTPUT}" MATCHES "\.(task)")
    list(APPEND VARS "--target-env" "spirv1.4")
  endif()

  if("${OUTPUT}" MATCHES "\.(16).")
    list(APPEND VARS "-DWORKGROUP_SIZE=16")
  endif()

  if("${OUTPUT}" MATCHES "\.(32).")
    list(APPEND VARS "-DWORKGROUP_SIZE=32")
  endif()

  if("${OUTPUT}" MATCHES "\.(64).")
    list(APPEND VARS "-DWORKGROUP_SIZE=64")
  endif()

  # debug info in shaders
  if(${CMAKE_BUILD_TYPE} MATCHES "Debug")
    list(APPEND VARS "-g")
  else()
    list(APPEND VARS "-g")
  endif()

  add_custom_command(
    OUTPUT     ${OUTPUT_FILE}
    DEPENDS    ${SOURCE_FILE} ${GLSL_SOURCES}
    COMMAND    ${CMAKE_COMMAND} -E make_directory "${PROJECT_BINARY_DIR}/sprv/"
    COMMAND    ${GLSLANGVALIDATOR} -V ${VARS} -I"${CMAKE_SOURCE_DIR}/shader" "${SOURCE_FILE}" -o ${OUTPUT_FILE}
    )
endfunction(add_shader)

# Ubershader flags:
#   MESH_TYPE  - type of mesh enum{landscape, object, skinned, morph, pfx}
#   SHADOW_MAP - output is shadowmap
#   FORWARD    - compute forward lighting
#   GBUFFER    - emit gbuffer
#   ATEST      - use alpha test
#   WATER      - water material
#   GHOST      - ghost material

function(add_geom_shader OUTPUT)
  list(SUBLIST ARGV 1 -1 VARS)

  add_shader("${OUTPUT}_slot.vert"      materials/main.vert  -S vert ${VARS})
  add_shader("${OUTPUT}_slot.mesh"      materials/main.vert  -S mesh ${VARS})
  add_shader("${OUTPUT}_bindless.vert"  materials/main.vert  -S vert ${VARS} -DBINDLESS)
  add_shader("${OUTPUT}_bindless.mesh"  materials/main.vert  -S mesh ${VARS} -DBINDLESS)

  set(OPENGOTHIC_SHADERS      ${OPENGOTHIC_SHADERS}      PARENT_SCOPE)
  set(OPENGOTHIC_SHADERS_SPRV ${OPENGOTHIC_SHADERS_SPRV} PARENT_SCOPE)
endfunction(add_geom_shader)

function(add_frag_shader OUTPUT)
  list(SUBLIST ARGV 1 -1 VARS)

  add_shader("${OUTPUT}_slot.frag"         materials/main.frag  ${VARS})
  add_shader("${OUTPUT}_bindless.frag"     materials/main.frag  ${VARS} -DBINDLESS)

  set(OPENGOTHIC_SHADERS      ${OPENGOTHIC_SHADERS}      PARENT_SCOPE)
  set(OPENGOTHIC_SHADERS_SPRV ${OPENGOTHIC_SHADERS_SPRV} PARENT_SCOPE)
endfunction(add_frag_shader)

# cluster culling
add_shader(cluster_init.comp            materials/cluster_init.comp)
add_shader(cluster_patch.comp           materials/cluster_patch.comp)
add_shader(visibility_pass.comp         materials/visibility_pass.comp)
add_shader(visibility_pass_hiz.comp     materials/visibility_pass.comp -DMAIN_VIEW -DHIZ)
add_shader(visibility_pass_hiz_cr.comp  materials/visibility_pass.comp -DMAIN_VIEW -DHIZ -DLARGE)

# mesh & vertex
add_geom_shader(main_lnd       "-DMESH_TYPE=0" -DFLAT_NORMAL)
add_geom_shader(main_lnd_f     "-DMESH_TYPE=0" -DFLAT_NORMAL -DFORWARD)
add_geom_shader(main_lnd_d     "-DMESH_TYPE=0" -DFLAT_NORMAL -DDEPTH_ONLY)
add_geom_shader(main_lnd_d_at  "-DMESH_TYPE=0" -DFLAT_NORMAL -DDEPTH_ONLY -DATEST)
add_geom_shader(main_lnd_v     "-DMESH_TYPE=0" -DFLAT_NORMAL -DDEPTH_ONLY -DVIRTUAL_SHADOW)
add_geom_shader(main_lnd_v_at  "-DMESH_TYPE=0" -DFLAT_NORMAL -DDEPTH_ONLY -DVIRTUAL_SHADOW -DATEST)

add_geom_shader(main_obj       "-DMESH_TYPE=1")
add_geom_shader(main_obj_f     "-DMESH_TYPE=1" -DFORWARD)
add_geom_shader(main_obj_d     "-DMESH_TYPE=1" -DDEPTH_ONLY)
add_geom_shader(main_obj_d_at  "-DMESH_TYPE=1" -DDEPTH_ONLY -DATEST)
add_geom_shader(main_obj_v     "-DMESH_TYPE=1" -DDEPTH_ONLY -DVIRTUAL_SHADOW)
add_geom_shader(main_obj_v_at  "-DMESH_TYPE=1" -DDEPTH_ONLY -DVIRTUAL_SHADOW -DATEST)

add_geom_shader(main_ani       "-DMESH_TYPE=2")
add_geom_shader(main_ani_f     "-DMESH_TYPE=2" -DFORWARD)
add_geom_shader(main_ani_d     "-DMESH_TYPE=2" -DDEPTH_ONLY)
add_geom_shader(main_ani_d_at  "-DMESH_TYPE=2" -DDEPTH_ONLY -DATEST)
add_geom_shader(main_ani_v     "-DMESH_TYPE=2" -DDEPTH_ONLY -DVIRTUAL_SHADOW)
add_geom_shader(main_ani_v_at  "-DMESH_TYPE=2" -DDEPTH_ONLY -DVIRTUAL_SHADOW -DATEST)

add_geom_shader(main_mph       "-DMESH_TYPE=3")
add_geom_shader(main_mph_f     "-DMESH_TYPE=3" -DFORWARD)
add_geom_shader(main_mph_d     "-DMESH_TYPE=3" -DDEPTH_ONLY)
add_geom_shader(main_mph_d_at  "-DMESH_TYPE=3" -DDEPTH_ONLY -DATEST)
add_geom_shader(main_mph_v     "-DMESH_TYPE=3" -DDEPTH_ONLY -DVIRTUAL_SHADOW)
add_geom_shader(main_mph_v_at  "-DMESH_TYPE=3" -DDEPTH_ONLY -DVIRTUAL_SHADOW -DATEST)

# tesselation
add_shader(main_lnd_f_slot.tesc      materials/main.tesc  "-DMESH_TYPE=0" -DWATER)
add_shader(main_lnd_f_slot.tese      materials/main.tese  "-DMESH_TYPE=0" -DWATER)
add_shader(main_lnd_f_bindless.tesc  materials/main.tesc  "-DMESH_TYPE=0" -DBINDLESS -DWATER)
add_shader(main_lnd_f_bindless.tese  materials/main.tese  "-DMESH_TYPE=0" -DBINDLESS -DWATER)

# materials landscape
add_frag_shader(main_lnd_g_s   -DFLAT_NORMAL -DGBUFFER -DSIMPLE_MAT)
add_frag_shader(main_lnd_g     -DFLAT_NORMAL -DGBUFFER)
add_frag_shader(main_lnd_g_at  -DFLAT_NORMAL -DGBUFFER -DATEST)
add_frag_shader(main_lnd_g_tx  -DFLAT_NORMAL -DGBUFFER)
add_frag_shader(main_lnd_f     -DFLAT_NORMAL -DFORWARD)
add_frag_shader(main_lnd_e     -DFLAT_NORMAL -DEMISSIVE)
add_frag_shader(main_lnd_x     -DFLAT_NORMAL -DGHOST)
add_frag_shader(main_lnd_w     -DFLAT_NORMAL -DWATER)
add_frag_shader(main_lnd_d     -DFLAT_NORMAL -DDEPTH_ONLY)
add_frag_shader(main_lnd_d_at  -DFLAT_NORMAL -DDEPTH_ONLY -DATEST)
add_frag_shader(main_lnd_v     -DFLAT_NORMAL -DDEPTH_ONLY -DVIRTUAL_SHADOW)
add_frag_shader(main_lnd_v_at  -DFLAT_NORMAL -DDEPTH_ONLY -DVIRTUAL_SHADOW -DATEST)

# materials objects
add_frag_shader(main_obj_g     -DGBUFFER)
add_frag_shader(main_obj_g_at  -DGBUFFER -DATEST)
add_frag_shader(main_obj_f     -DFORWARD)
add_frag_shader(main_obj_e     -DEMISSIVE)
add_frag_shader(main_obj_w     -DWATER)
add_frag_shader(main_obj_x     -DGHOST)
add_frag_shader(main_obj_d     -DDEPTH_ONLY)
add_frag_shader(main_obj_d_at  -DDEPTH_ONLY -DATEST)
add_frag_shader(main_obj_v     -DDEPTH_ONLY -DVIRTUAL_SHADOW)
add_frag_shader(main_obj_v_at  -DDEPTH_ONLY -DVIRTUAL_SHADOW -DATEST)

# pfx geometry
add_shader(main_pfx.vert       materials/pfx.vert   "-DMESH_TYPE=4" -DVT_COLOR)
add_shader(main_pfx_f.vert     materials/pfx.vert   "-DMESH_TYPE=4" -DFORWARD    -DVT_COLOR)
add_shader(main_pfx_d.vert     materials/pfx.vert   "-DMESH_TYPE=4" -DDEPTH_ONLY -DVT_COLOR)
add_shader(main_pfx_d_at.vert  materials/pfx.vert   "-DMESH_TYPE=4" -DDEPTH_ONLY -DVT_COLOR -DATEST)
add_shader(main_pfx_v.vert     materials/pfx.vert   "-DMESH_TYPE=4" -DDEPTH_ONLY -DVIRTUAL_SHADOW -DVT_COLOR)
add_shader(main_pfx_v_at.vert  materials/pfx.vert   "-DMESH_TYPE=4" -DDEPTH_ONLY -DVIRTUAL_SHADOW -DVT_COLOR -DATEST)
# materials pfx
add_shader(main_pfx_g.frag     materials/main.frag  "-DMESH_TYPE=4" -DGBUFFER    -DVT_COLOR)
add_shader(main_pfx_g_at.frag  materials/main.frag  "-DMESH_TYPE=4" -DGBUFFER    -DVT_COLOR -DATEST)
add_shader(main_pfx_f.frag     materials/main.frag  "-DMESH_TYPE=4" -DFORWARD    -DVT_COLOR)
add_shader(main_pfx_f_at.frag  materials/main.frag  "-DMESH_TYPE=4" -DFORWARD    -DVT_COLOR -DATEST)
add_shader(main_pfx_e.frag     materials/main.frag  "-DMESH_TYPE=4" -DEMISSIVE   -DVT_COLOR)
add_shader(main_pfx_e_at.frag  materials/main.frag  "-DMESH_TYPE=4" -DEMISSIVE   -DVT_COLOR -DATEST)
add_shader(main_pfx_x.frag     materials/main.frag  "-DMESH_TYPE=4" -DGHOST      -DVT_COLOR)
add_shader(main_pfx_x_at.frag  materials/main.frag  "-DMESH_TYPE=4" -DGHOST      -DVT_COLOR -DATEST)
add_shader(main_pfx_d.frag     materials/main.frag  "-DMESH_TYPE=4" -DDEPTH_ONLY -DVT_COLOR)
add_shader(main_pfx_d_at.frag  materials/main.frag  "-DMESH_TYPE=4" -DDEPTH_ONLY -DVT_COLOR -DATEST)
add_shader(main_pfx_v.frag     materials/main.frag  "-DMESH_TYPE=4" -DDEPTH_ONLY -DVIRTUAL_SHADOW -DVT_COLOR)
add_shader(main_pfx_v_at.frag  materials/main.frag  "-DMESH_TYPE=4" -DDEPTH_ONLY -DVIRTUAL_SHADOW -DVT_COLOR -DATEST)


add_shader(item.vert                 inventory/item.vert)
add_shader(item.frag                 inventory/item.frag)

add_shader(stash.frag                stash.frag)

add_shader(direct_light.frag         lighting/direct_light.frag)
add_shader(direct_light_sh.frag      lighting/direct_light.frag -DSHADOW_MAP)
add_shader(direct_light_rq.frag      lighting/direct_light.frag -DSHADOW_MAP -DRAY_QUERY -DRAY_QUERY_AT)
add_shader(direct_light_vsm.frag     virtual_shadow/vsm_direct_light.frag)

add_shader(ambient_light.frag        lighting/ambient_light.frag)
add_shader(ambient_light_ssao.frag   lighting/ambient_light.frag -DSSAO)

add_shader(light.vert                lighting/light.vert)
add_shader(light.frag                lighting/light.frag)
add_shader(light_rq.frag             lighting/light.frag -DRAY_QUERY)
add_shader(light_rq_at.frag          lighting/light.frag -DRAY_QUERY -DRAY_QUERY_AT)
add_shader(light_vsm.frag            lighting/light.frag -DVIRTUAL_SHADOW)

add_shader(irradiance.comp           lighting/irradiance.comp)
add_shader(sky_exposure.comp         lighting/sky_exposure.comp)

add_shader(tonemapping.frag          lighting/tonemapping.frag)
add_shader(tonemapping_up.frag       lighting/tonemapping.frag -DUPSCALE)

add_shader(water_reflection.mesh     water/water_reflection.mesh)
add_shader(water_reflection.frag     water/water_reflection.frag)
add_shader(water_reflection_ssr.frag water/water_reflection.frag -DSSR)

add_shader(underwater_t.frag         water/underwater.frag "")
add_shader(underwater_s.frag         water/underwater.frag -DSCATTERING)

# Based on Scalable and Production Ready Sky and Atmosphere
# https://sebh.github.io/publications/egsr2020.pdf
# https://www.shadertoy.com/view/slSXRW
add_shader(sky_transmittance.frag    sky/sky_transmittance.frag)
add_shader(sky_multi_scattering.frag sky/sky_multi_scattering.frag)
add_shader(sky_view_lut.frag         sky/sky_view_lut.frag)
add_shader(sky_view_clouds_lut.frag  sky/sky_view_clouds_lut.frag)

add_shader(clouds_lut.comp           sky/clouds_lut.comp)
add_shader(fog_view_lut.comp         sky/fog_view_lut.comp -DSUN_SCATTERING -DMULTI_SCATTERING)
add_shader(fog_view_lut_sep.comp     sky/fog_view_lut.comp -DSUN_SCATTERING -DMULTI_SCATTERING -DSEPARABLE)

add_shader(sky.frag                  sky/sky.frag)
add_shader(sky_sep.frag              sky/sky.frag -DSEPARABLE)
add_shader(fog3d.comp                sky/fog.frag -S comp -DVOLUMETRIC)
add_shader(fog3d_hq.frag             sky/fog.frag -DVOLUMETRIC)
add_shader(fog_epipolar.frag         sky/fog_epipolar.frag)
add_shader(fog.frag                  sky/fog.frag)
#
add_shader(fog_epipolar_vsm.comp       epipolar/fog_epipolar.comp)
add_shader(fog_epipolar_occlusion.comp epipolar/fog_epipolar_occlusion.comp)

add_shader(sky_pathtrace.frag        sky/sky_pathtrace.frag)

add_shader(sun.vert                  sky/sun.vert)
add_shader(sun.frag                  sky/sun.frag)

# copy/edit
add_shader(triangle.vert             triangle.vert)
add_shader(triangle_uv.vert          triangle.vert -DHAS_UV)
add_shader(copy.frag                 copy.frag)
add_shader(copy.comp                 copy.comp)
add_shader(copy_img.comp             copy_img.comp)
add_shader(patch.comp                patch.comp)

add_shader(ssao.comp                 ssao/ssao.comp)
add_shader(ssao_blur.comp            ssao/ssao_blur.comp)

# GPU-driven: HiZ
if(IOS)
  add_shader(hiz_mip.comp            hiz/hiz_mip.comp -DIOS)
else()
  add_shader(hiz_mip.comp            hiz/hiz_mip.comp)
endif()
add_shader(hiz_pot.comp              hiz/hiz_pot.comp)

# Global illumination
add_shader(probe_dbg.vert            lighting/rt/probe_dbg.vert)
add_shader(probe_dbg.frag            lighting/rt/probe_dbg.frag)

add_shader(probe_hit_dbg.vert        lighting/rt/probe_hit_dbg.vert)
add_shader(probe_hit_dbg.frag        lighting/rt/probe_hit_dbg.frag)

add_shader(probe_init.comp           lighting/rt/probe_aux.comp        -DINIT)
add_shader(probe_clear.comp          lighting/rt/probe_aux.comp        -DCLEAR)
add_shader(probe_clear_hash.comp     lighting/rt/probe_aux.comp        -DCLEAR_HASH)
add_shader(probe_make_hash.comp      lighting/rt/probe_aux.comp        -DMAKE_HASH)
add_shader(probe_prune.comp          lighting/rt/probe_aux.comp        -DPRUNE)
add_shader(probe_vote.comp           lighting/rt/probe_allocation.comp -DVOTE)
add_shader(probe_allocation.comp     lighting/rt/probe_allocation.comp -DALLOC)
add_shader(probe_trace.comp          lighting/rt/probe_trace.comp      -DRAY_QUERY -DRAY_QUERY_AT)
add_shader(probe_lighting.comp       lighting/rt/probe_lighting.comp)

add_shader(probe_ambient.vert        triangle.vert)
add_shader(probe_ambient.frag        lighting/rt/probe_ambient.frag)

# CMAA2
add_shader(cmaa2_edges_color2x2_quality_0.comp     antialiasing/cmaa2/edge_color2x2.comp -DCMAA2_STATIC_QUALITY_PRESET=0)
add_shader(cmaa2_edges_color2x2_quality_1.comp     antialiasing/cmaa2/edge_color2x2.comp -DCMAA2_STATIC_QUALITY_PRESET=1)

add_shader(cmaa2_process_candidates.comp           antialiasing/cmaa2/process_candidates.comp)
add_shader(cmaa2_deferred_color_apply_2x2.vert     antialiasing/cmaa2/deferred_color_apply_2x2.vert)
add_shader(cmaa2_deferred_color_apply_2x2.frag     antialiasing/cmaa2/deferred_color_apply_2x2.frag)

# virtual shadows
add_shader(vsm_dbg.frag              virtual_shadow/vsm_direct_light.frag -DDEBUG)
add_shader(vsm_mark_pages.comp       virtual_shadow/vsm_mark_pages.comp)
add_shader(vsm_clear.comp            virtual_shadow/vsm_clear.comp)
add_shader(vsm_clear_omni.comp       virtual_shadow/vsm_clear_omni.comp)
add_shader(vsm_trim_pages.comp       virtual_shadow/vsm_trim_pages.comp)
add_shader(vsm_clump_pages.comp      virtual_shadow/vsm_clump_pages.comp)
add_shader(vsm_list_pages.comp       virtual_shadow/vsm_list_pages.comp)
add_shader(vsm_sort_pages.comp       virtual_shadow/vsm_sort_pages.comp)
add_shader(vsm_alloc_pages.comp      virtual_shadow/vsm_alloc_pages.comp)
add_shader(vsm_alloc_pages2.comp     virtual_shadow/vsm_alloc_pages2.comp)
add_shader(vsm_merge_pages.comp      virtual_shadow/vsm_merge_pages.comp)
add_shader(vsm_pack_draws0.comp      virtual_shadow/vsm_pack_draws.comp   -DPASS0)
add_shader(vsm_pack_draws1.comp      virtual_shadow/vsm_pack_draws.comp   -DPASS1)
add_shader(vsm_visibility_pass.comp  virtual_shadow/vsm_visibility_pass.comp -DVIRTUAL_SHADOW)
# virtual shadows: omni lights
add_shader(vsm_cull_lights.comp      virtual_shadow/vsm_cull_lights.comp)
add_shader(vsm_mark_omni_pages.comp  virtual_shadow/vsm_mark_omni_pages.comp)
add_shader(vsm_postprocess_omni.comp virtual_shadow/vsm_postprocess_omni.comp)
# virtual shadows: fog
add_shader(vsm_fog_shadow.comp       virtual_shadow/vsm_fog_shadow.comp)
add_shader(vsm_fog_mark_pages.comp   virtual_shadow/vsm_fog_shadow.comp -DPAGES)
add_shader(vsm_fog_trace.comp        virtual_shadow/vsm_fog_trace.comp)

add_shader(vsm_rendering.comp        virtual_shadow/vsm_rendering.comp)

# software meshlet rt
add_shader(rtsm_clear.comp           rtsm/rtsm_clear.comp)
add_shader(rtsm_mark_pages.comp      rtsm/rtsm_mark_pages.comp)
add_shader(rtsm_mark_fog_pages.comp  rtsm/rtsm_mark_fog_pages.comp)
add_shader(rtsm_hiz_pages.comp       rtsm/rtsm_hiz_pages.comp)
add_shader(rtsm_culling.comp         rtsm/rtsm_culling.comp)
add_shader(rtsm_position.comp        rtsm/rtsm_position.comp)

add_shader(rtsm_meshlet_cull.comp    rtsm/rtsm_meshlet_cull.comp)
add_shader(rtsm_meshlet_complex.comp rtsm/rtsm_meshlet_complex.comp)
add_shader(rtsm_sample_cull.comp     rtsm/rtsm_sample_cull.comp)
add_shader(rtsm_primitive_cull.comp  rtsm/rtsm_primitive_cull.comp)

add_shader(rtsm_raster.comp          rtsm/rtsm_raster.comp)
add_shader(rtsm_direct_light.frag    rtsm/rtsm_direct_light.frag)

add_shader(rtsm_rendering.comp       rtsm/rtsm_rendering.comp)
add_shader(rtsm_omni_rendering.comp  rtsm/rtsm_rendering.comp)
add_shader(rtsm_dbg.frag             rtsm/rtsm_dbg.frag)

add_shader(rtsm_omni_clear.comp      rtsm/rtsm_omni_clear.comp)
add_shader(rtsm_cull_lights.comp     rtsm/rtsm_cull_lights.comp)
add_shader(rtsm_omni_lights.comp     rtsm/rtsm_omni_lights.comp)
add_shader(rtsm_omni_bboxes.comp     rtsm/rtsm_omni_bboxes.comp)
add_shader(rtsm_omni_culling.comp    rtsm/rtsm_omni_culling.comp)
add_shader(rtsm_omni_position.comp   rtsm/rtsm_position.comp -DOMNI)
add_shader(rtsm_omni_meshlet.comp    rtsm/rtsm_omni_meshlet.comp)
add_shader(rtsm_omni_backface.comp   rtsm/rtsm_omni_backface.comp)
add_shader(rtsm_omni_compact.comp    rtsm/rtsm_omni_compact.comp)
add_shader(rtsm_omni_primitive.comp  rtsm/rtsm_omni_primitive.comp)
add_shader(rtsm_omni_raster.comp     rtsm/rtsm_omni_raster.comp)

# software rendering
add_shader(sw_rendering_imm.comp  software_rendering/sw_rendering_imm.comp)
add_shader(sw_rendering_tbr.comp  software_rendering/sw_rendering_tbr.comp)
add_shader(sw_light.comp          software_rendering/sw_light.comp)
add_shader(vbuffer_blit.frag      software_rendering/vbuffer_blit.frag)

add_custom_command(
  OUTPUT     ${HEADER} ${CPP}
  DEPENDS    ${OPENGOTHIC_SHADERS_SPRV}
  COMMAND    ${CMAKE_COMMAND} -E make_directory "${PROJECT_BINARY_DIR}/sprv/"
  COMMAND    ${CMAKE_COMMAND} -P "${CMAKE_SOURCE_DIR}/shader/link_shaders.cmake"
  WORKING_DIRECTORY "${PROJECT_BINARY_DIR}"
  )

add_custom_target(shader
  DEPENDS ${HEADER} ${CPP}
  SOURCES ${SOURCES} "${CMAKE_SOURCE_DIR}/shader/link_shaders.cmake")

add_library(${PROJECT_NAME} STATIC ${HEADER} ${CPP})
add_dependencies(${PROJECT_NAME} shader)

target_include_directories(${PROJECT_NAME} PUBLIC "${CMAKE_CURRENT_BINARY_DIR}/sprv")
set_target_properties(
    ${PROJECT_NAME} PROPERTIES
    PUBLIC_HEADER ${HEADER}
    )
