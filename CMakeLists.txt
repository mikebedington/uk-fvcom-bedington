cmake_minimum_required(VERSION 2.8)

project(FVCOM Fortran C)

set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} "${CMAKE_CURRENT_SOURCE_DIR}/cmake/Modules/")

set(ROOT "${CMAKE_CURRENT_SOURCE_DIR}/FVCOM_source")

find_path(JULIAN_BASE fjulian.inc HINTS "${ROOT}/libs/julian" DOC "Source directory of julian")
find_path(METIS_BASE metis.h HINTS "${ROOT}/libs/metis" DOC "Source directory of METIS")

include_directories(${ROOT} ${JULIAN_BASE})

add_definitions(-DLIMITED_2 -DGCY1 -DMPDATA)

option(FVCOM_USE_FABM "Enable FABM" OFF)
if(FVCOM_USE_FABM)
  add_definitions(-DFABM)
endif()

option(FVCOM_USE_DOUBLE_PRECISION "Use double precision" OFF)
if(FVCOM_USE_DOUBLE_PRECISION)
  add_definitions(-DDOUBLE_PRECISION)
  set(FABM_REAL_KIND "SELECTED_REAL_KIND(12)")
  option(FVCOM_USE_SINGLE_OUTPUT "Use single precision for output" OFF)
  if(FVCOM_USE_DOUBLE_PRECISION)
    add_definitions(-DSINGLE_OUTPUT)
  endif()
else()
  set(FABM_REAL_KIND "SELECTED_REAL_KIND(6)")
endif()

option(FVCOM_USE_WET_DRY "Enable wetting-drying" ON)
if(FVCOM_USE_WET_DRY)
  add_definitions(-DWET_DRY)
endif()

option(FVCOM_USE_MULTIPROCESSOR "Enable multi-processor use" OFF)
if(FVCOM_USE_MULTIPROCESSOR)
  find_package(MPI REQUIRED)
  if(MPI_Fortran_INCLUDE_PATH)
    include_directories("${MPI_Fortran_INCLUDE_PATH}")
  endif()
  add_definitions(-DMULTIPROCESSOR)
endif()

# Customize compiler flags
if(${CMAKE_Fortran_COMPILER_ID} STREQUAL "GNU")
  add_definitions(-DGFORTRAN)
elseif(${CMAKE_Fortran_COMPILER_ID} STREQUAL "Cray")
elseif(${CMAKE_Fortran_COMPILER_ID} STREQUAL "Intel")
  if(WIN32)
    set(CMAKE_Fortran_FLAGS_DEBUG "${CMAKE_Fortran_FLAGS_DEBUG} /Od")
    set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} /STACK:134217728")
  endif()
endif()

find_package(NetCDF REQUIRED)
include_directories("${NetCDF_INCLUDE_DIRS}")
if (NetCDF_STATIC_MSVC_BUILD)
  # On Windows with a statically-compiled NetCDF library - compile all code against static runtime.
  # This MUST be done before any targets are added.
  set(CMAKE_Fortran_FLAGS "${CMAKE_Fortran_FLAGS} /libs:static")
  set(CMAKE_C_FLAGS_RELEASE "${CMAKE_C_FLAGS_RELEASE} /MT")
  set(CMAKE_C_FLAGS_DEBUG "${CMAKE_C_FLAGS_DEBUG} /MTd")
endif()

add_library(metis
  ${METIS_BASE}/coarsen.c
  ${METIS_BASE}/fm.c
  ${METIS_BASE}/initpart.c
  ${METIS_BASE}/match.c
  ${METIS_BASE}/ccgraph.c
  ${METIS_BASE}/memory.c
  ${METIS_BASE}/pmetis.c
  ${METIS_BASE}/pqueue.c
  ${METIS_BASE}/refine.c
  ${METIS_BASE}/util.c
  ${METIS_BASE}/timing.c
  ${METIS_BASE}/debug.c
  ${METIS_BASE}/bucketsort.c
  ${METIS_BASE}/graph.c
  ${METIS_BASE}/stat.c
  ${METIS_BASE}/kmetis.c
  ${METIS_BASE}/kwayrefine.c
  ${METIS_BASE}/kwayfm.c
  ${METIS_BASE}/balance.c
  ${METIS_BASE}/ometis.c
  ${METIS_BASE}/srefine.c
  ${METIS_BASE}/sfm.c
  ${METIS_BASE}/separator.c
  ${METIS_BASE}/mincover.c
  ${METIS_BASE}/mmd.c
  ${METIS_BASE}/mesh.c
  ${METIS_BASE}/meshpart.c
  ${METIS_BASE}/frename.c
  ${METIS_BASE}/fortran.c
  ${METIS_BASE}/myqsort.c
  ${METIS_BASE}/compress.c
  ${METIS_BASE}/parmetis.c
  ${METIS_BASE}/estmem.c
  ${METIS_BASE}/mpmetis.c
  ${METIS_BASE}/mcoarsen.c
  ${METIS_BASE}/mmatch.c
  ${METIS_BASE}/minitpart.c
  ${METIS_BASE}/mbalance.c
  ${METIS_BASE}/mrefine.c
  ${METIS_BASE}/mutil.c
  ${METIS_BASE}/mfm.c
  ${METIS_BASE}/mkmetis.c
  ${METIS_BASE}/mkwayrefine.c
  ${METIS_BASE}/mkwayfmh.c
  ${METIS_BASE}/mrefine2.c
  ${METIS_BASE}/minitpart2.c
  ${METIS_BASE}/mbalance2.c
  ${METIS_BASE}/mfm2.c
  ${METIS_BASE}/kvmetis.c
  ${METIS_BASE}/kwayvolrefine.c
  ${METIS_BASE}/kwayvolfm.c
  ${METIS_BASE}/subdomains.c
  ${METIS_BASE}/partition.c
)
target_include_directories(metis PRIVATE ${METIS_BASE})
if(MSVC)
  target_compile_definitions(metis PRIVATE __VC__)
endif()

add_library(julian
   ${JULIAN_BASE}/dates.c
   ${JULIAN_BASE}/format.c
   ${JULIAN_BASE}/juldates.c
   ${JULIAN_BASE}/leapsecs.c
   ${JULIAN_BASE}/parse.c
   ${JULIAN_BASE}/seconds.c
   ${JULIAN_BASE}/tai_et.c
   ${JULIAN_BASE}/utc_tai.c
   ${JULIAN_BASE}/fortran.c
   ${JULIAN_BASE}/rlerrors.c
   ${JULIAN_BASE}/rlmemory.c
)

add_library(fjulian
   ${JULIAN_BASE}/fjulian.for
   ${JULIAN_BASE}/fstrings.for
)

if(FVCOM_USE_FABM)
  find_path(FABM_BASE src/fabm.F90 DOC "Path to FABM source directory.")
  set(FABM_FORCED_HOST fvcom)
  add_subdirectory(${FABM_BASE}/src fabm)
  set(EXTRA_SOURCES ${EXTRA_SOURCES} ${ROOT}/mod_fabm_3D.F ${ROOT}/mod_fabm_data.F)
endif()

add_library(FVCOM_C
   ${ROOT}/func_pointer.c
)

if(FVCOM_USE_HEATING_CALCULATED)
  set(EXTRA_SOURCES ${EXTRA_SOURCES} ${ROOT}/mod_heatflux.F)
endif()
if(FVCOM_USE_HEATING_CALCULATED_GL)
  set(EXTRA_SOURCES ${EXTRA_SOURCES} ${ROOT}/mod_heatflux_gl.F)
endif()

add_executable(FVCOM_exe
${ROOT}/adcor.F
${ROOT}/adjust2d3d.F
${ROOT}/adjust_ts.F
${ROOT}/advave_edge_gcn.F
${ROOT}/advave_edge_gcy.F
${ROOT}/advection_edge_gcn.F
${ROOT}/advection_edge_gcy.F
${ROOT}/adv_q.F
${ROOT}/adv_s.F
${ROOT}/adv_t.F
${ROOT}/adv_uv_edge_gcn.F
${ROOT}/adv_uv_edge_gcy.F
${ROOT}/allocate_all.F
${ROOT}/baropg.F
${ROOT}/bcond_gcn.F
${ROOT}/bcond_gcy.F
${ROOT}/bcond_ts.F
${ROOT}/brough.F
${ROOT}/calc_vort.F
${ROOT}/cell_area.F
${ROOT}/cntrl_prmtrs.F
${ROOT}/conv_over.F
${ROOT}/coords_n_const.F
${ROOT}/depth_check.F
${ROOT}/depth_grad.F
${ROOT}/edge_len.F
${ROOT}/enkf_ncdio.F
${ROOT}/eqs_of_state.F
${ROOT}/extelpf_edge.F
${ROOT}/extel_edge.F
${ROOT}/external_step.F
${ROOT}/extuv_edge.F
${ROOT}/fct_q2.F
${ROOT}/fct_q2l.F
${ROOT}/fct_s.F
${ROOT}/fct_t.F
${ROOT}/fvcom.F
${ROOT}/genmap.F
${ROOT}/genmap_lsf.F
${ROOT}/genmap_obc.F
${ROOT}/ghostuv.F
${ROOT}/grid_metrics.F
${ROOT}/ice_albedo.F
${ROOT}/ice_atmo.F
${ROOT}/ice_calendar.F
${ROOT}/ice_constants.F
#${ROOT}/ice_coupling.F
${ROOT}/ice_domain.F
${ROOT}/ice_fileunits.F
${ROOT}/ice_flux.F
${ROOT}/ice_flux_in.F
${ROOT}/ice_grid.F
${ROOT}/ice_init.F
${ROOT}/ice_itd.F
${ROOT}/ice_itd_linear.F
${ROOT}/ice_kinds_mod.F
${ROOT}/ice_mechred.F
${ROOT}/ice_model_size.F
${ROOT}/ice_ocean.F
${ROOT}/ice_scaling.F
${ROOT}/ice_state.F
${ROOT}/ice_therm_itd.F
${ROOT}/ice_therm_vertical.F
${ROOT}/ice_work.F
${ROOT}/icing.F
${ROOT}/init_sed.F
${ROOT}/internal_step.F
${ROOT}/linklist.F
${ROOT}/load_grid.F
${ROOT}/longshore_flow.F
${ROOT}/mod_action_ex.F
${ROOT}/mod_action_im.F
${ROOT}/mod_assim.F
${ROOT}/mod_balance_2d.F
${ROOT}/mod_bbl.F
${ROOT}/mod_bio_3D.F
${ROOT}/mod_boundschk.F
${ROOT}/mod_bulk.F
${ROOT}/mod_clock.F
${ROOT}/mod_dam.F
${ROOT}/mod_dye.F
${ROOT}/mod_enkf.F
${ROOT}/mod_enkfassim.F
${ROOT}/mod_enkf_ncd.F
${ROOT}/mod_enkf_obs.F
${ROOT}/mod_etkf.F
${ROOT}/mod_force.F
${ROOT}/mod_gotm.F
${ROOT}/mod_ice.F
${ROOT}/mod_ice2d.F
${ROOT}/mod_input.F
${ROOT}/mod_interp.F
${ROOT}/mod_lag.F
${ROOT}/mod_main.F
${ROOT}/mod_main_wave.F
${ROOT}/mod_meanflow.F
${ROOT}/mod_ncdio.F
${ROOT}/mod_ncll.F
${ROOT}/mod_nctools.F
${ROOT}/mod_nesting.F
${ROOT}/mod_newinp.F
${ROOT}/mod_non_hydro.F
${ROOT}/mod_northpole.F
${ROOT}/mod_obcs.F
${ROOT}/mod_obcs2.F
${ROOT}/mod_obcs3.F
${ROOT}/mod_onedtide.F
${ROOT}/mod_optimal_interpolation.F
${ROOT}/mod_par.F
${ROOT}/mod_petsc.F
${ROOT}/mod_plbc.F
${ROOT}/mod_prec.F
${ROOT}/mod_probe.F
${ROOT}/mod_pwp.F
${ROOT}/mod_report.F
${ROOT}/mod_rrk.F
${ROOT}/mod_rrkassim.F
${ROOT}/mod_rrkf_obs.F
${ROOT}/mod_scal.F
${ROOT}/mod_sed.F
${ROOT}/mod_semi_implicit.F
${ROOT}/mod_setup.F
${ROOT}/mod_set_time.F
${ROOT}/mod_sng.F
${ROOT}/mod_solar.F
${ROOT}/mod_spherical.F
${ROOT}/mod_startup.F
${ROOT}/mod_station_timeseries.F
${ROOT}/mod_time.F
${ROOT}/mod_tridiag.F
${ROOT}/mod_types.F
${ROOT}/mod_utils.F
${ROOT}/mod_visit.F
${ROOT}/mod_wavesetup.F
${ROOT}/mod_wave_current_interaction.F
${ROOT}/mod_wd.F
${ROOT}/mod_wqm.F
${ROOT}/namelist.F
${ROOT}/nh_set_nesting.F
${ROOT}/ocpcre.F
${ROOT}/ocpids.F
${ROOT}/ocpmix.F
${ROOT}/open_all.F
${ROOT}/particle.F
${ROOT}/phy_baropg.F
${ROOT}/print_vals.F
${ROOT}/rho_pmean.F
#${ROOT}/sectinf.F
${ROOT}/setup_domain.F
${ROOT}/shape_coef_gcn.F
${ROOT}/shape_coef_gcy.F
${ROOT}/sinter.F
${ROOT}/startup_type.F
${ROOT}/swancom1.F
${ROOT}/swancom2.F
${ROOT}/swancom3.F
${ROOT}/swancom4.F
${ROOT}/swancom5.F
${ROOT}/swanmain.F
${ROOT}/swanpre1.F
${ROOT}/swanpre2.F
${ROOT}/swanser.F
${ROOT}/swmod1.F
${ROOT}/swmod2.F
${ROOT}/swmod3.F
${ROOT}/tge.F
${ROOT}/vdif_q.F
${ROOT}/vdif_ts.F
${ROOT}/vdif_ts_gom.F
${ROOT}/vdif_uv.F
${ROOT}/vertvl_edge.F
${ROOT}/viscofh.F
${ROOT}/visitsim.F
${ROOT}/wreal.F
${EXTRA_SOURCES}
${MPI_Fortran_EXTRA_SOURCES}
)
set_property(TARGET FVCOM_exe PROPERTY Fortran_FORMAT FREE)
target_link_libraries(FVCOM_exe julian fjulian metis)
target_link_libraries(FVCOM_exe FVCOM_C)
target_link_libraries(FVCOM_exe "${NetCDF_LIBRARIES}")

if(FVCOM_USE_FABM)
  target_include_directories(FVCOM_exe PRIVATE ${CMAKE_CURRENT_BINARY_DIR}/fabm/modules)
  target_link_libraries(FVCOM_exe fabm)
endif()

if (NetCDF_STATIC_MSVC_BUILD)
  set_property(TARGET FVCOM_exe PROPERTY LINK_FLAGS_DEBUG "/NODEFAULTLIB:\"libcmt\"")
endif()
if(FVCOM_USE_MULTIPROCESSOR AND MPI_Fortran_LIBRARIES)
  target_link_libraries(FVCOM_exe ${MPI_Fortran_LIBRARIES})
endif()
