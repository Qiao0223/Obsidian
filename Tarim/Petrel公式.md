# 1. 基本公式
```
//Vp拟合Vs
Vs=Sqrt(0.19*Vp*Vp+2965899)

//Vp转DT
DT = 304800/Vp

//DT拟合密度
Den = -0.0071*DT+3.0294

//Vp与Vs比值
VpVs = Vp/Vs

//泊松比
PR = (0.5*VpVs*VpVs-1)/(VpVs*VpVs-1)

//横波阻抗Z(m/s*kg/m3) 
Zs = Den*Vs*1000

//杨氏模量E(Gpa)
E = (2*Zs*Zs)*(1+PR)/(Den*1000)/1000000000

//趋势声波时差
DT_trend = 133.5+(-0.00932741*(-Z+start))

//静水压力(Mpa)
Phydro = 1000*9.8*(-Z+start)/1000000

//孔隙压力
Pp = Pv-(Pv-Phydro)*Pow(DT_trend/DT,0.02)

//最大水平地应力
sigma_1 = Pp*0.5
sigma_2 = (PR/(1-PR))*(Pv-sigma_1)
sigma_3 = (E/(1-PR*PR))*(k1+PR*k2)

Shmax = (PR/(1-PR))*(Pv-1*Pp)+(E/(1-PR*PR))*(k1+PR*k2)+Pp

E_temp = If (E > 100, 0, If (E < 15, 0, E))
E_mod = E_temp*0.5533+1.1499

Pp=Pv_mod-(Pv_mod-Phydro)*Pow( DT_trend/DT, 2)

Shmax_temp=(PR/(1-PR))*(Pv_mod-1*Pp)+(E_mod*10/(1-PR*PR))*(k1+PR*k2)
Shmax_temp = (PR/(1-PR))*(Pv_mod-1*Pp)+(E_mod/(1-PR*PR))*(k1+PR*k2)
Pp = Pv_mod-(Pv_mod-Phydro)*Pow( DT_trend/DT,0.01)
Shmax_temp = (PR/(1-PR))*(Pv_mod-1*Pp)+(E_mod/(1-PR*PR))*(k1+PR*k2)
Shmax_temp = (PR/(1-PR))*(Pv_mod-1*Pp)+(E_mod/(1-PR*PR))*(k1+PR*k2)
Shmax_temp = (PR/(1-PR))*(Pv_mod-1*Pp)+(E_mod/(1-PR*PR))*(k1+PR*k2)+Pp
Shmax_temp = (PR/(1-PR))*(Pv_mod-1*Pp)-(E_mod/(1-PR*PR))*(k1+PR*k2)+Pp
Shmax_temp = (PR/(1-PR))*(Pv_mod-1*Pp)-(E_mod/(1-PR*PR))*(k1+PR*k2)+1.5*Pp
Shmax_temp = (PR/(1-PR))*(Pv_mod-1*Pp)-(E_mod/(1-PR*PR))*(k1+PR*k2)+1.2*Pp
Shmax_temp = (PR/(1-PR))*(Pv_mod-1*Pp)+1.2*Pp
Shmax_1 = (PR/(1-PR))*(Pv_mod-1*Pp)
Shmax_2 = Pp
Shmax_3 = (E_mod/(1-PR*PR))*(k1+PR*k2)
SHmin = SH_1+SH_2+SH_k_max
SHmax = SH_1+SH_2+Sh_k_min
SHmin = SH_1+SH_2-SH_k_max
a = SH_1+SH_2
H1_H2_mod = If (H1_H2 > 200, 0, If (H1_H2 < 50, 0, H1_H2))
H1_H2_mod = If (H1_H2 > 200, 50, If (H1_H2 < 50, 50, H1_H2))
SHmax = SH_1+SH_2+SH_k_max
SHmax = SH_1+1.2*SH_2+SH_k_max
SHmax = SH_1+1.5*SH_2+SH_k_max
SHmax = SH_1+1.5*SH_2+0.5*SH_k_max
SHmax = SH_1+1.5*SH_2+0.2*SH_k_max
SHmax = SH_1+1.5*SH_2+0.3*SH_k_max
SHmax = SH_1+1.5*SH_2+Pow( SH_k_max,0.01 )
SHmax = SH_1+1.5*SH_2+Pow( SH_k_max,0.1 )
SHmax = SH_1+1.5*SH_2+0.3*SH_k_max
E_mod = If (E_temp > 150, 0, If (E_temp < 0, 0, E_temp))
E_mod = If (E_temp > 100, 0, If (E_temp < 0, 0, E_temp))
Pp = Pv_mod-(Pv_mod-Phydro)*Pow( DT_trend/DT,0.01)
SHmax = SH_1+1.5*SH_2+0.1*SH_k_max
SHmax = Pv_mod-(Pv_mod-Phydro)*Pow( DT_trend/DT,0.01)-20
SHmax = Pv_mod-(Pv_mod-Phydro)*Pow( DT_trend/DT,0.01)
SHmax = SH_1+1.5*SH_2+0.1*SH_k_max
SHmax = SH_1+1.5*SH_2+0.1*SH_k_max-20
SHmax = SH_1+1.5*SH_2-0.1*SH_k_max
SHmax = SH_1+1.5*SH_2-0.1*SH_k_max-10
SHmax = SH_1+1.5*SH_2-0.1*SH_k_max-15
SHmax = SH_1+1.5*SH_2-0.15*SH_k_max-15
SHmax = SH_1+1.5*SH_2-0.15*SH_k_max-13
SHmax = SH_1+1.5*SH_2-0.15*SH_k_max-15
SHmin = SH_1+SH_2-Sh_k_min
SHmin = SH_1+SH_2-0.2*Sh_k_min
SHmin = SH_1+SH_2-0.2*Sh_k_min+5
SHmin = SH_1+SH_2-0.15*Sh_k_min
SHmin = SH_1+SH_2-0.15*Sh_k_min+5
SHmin = SH_1+SH_2-0.15*Sh_k_min+2
SHmin = SH_1+SH_2-0.10*Sh_k_min+2
SHmin = SH_1+SH_2-0.3*Sh_k_min+2
SHmin = SH_1+SH_2-0.3*Sh_k_min
SHmin = SH_1+SH_2-0.15*Sh_k_min+2
a = fault_diffusion_crop * (coherent_energy+energy_gradient_magnitude+energy_ratio_similarity+outer_product_similarity+sobel_filter_similarity)+k1
a = fault_diffusion_crop*(coherent_energy+energy_gradient_magnitude)
a = fault_diffusion_crop*(sobel_filter_similarity+energy_gradient_magnitude)
a = fault_diffusion_crop*(sobel_filter_similarity+energy_gradient_magnitude)+k1
```

# 2. 数据过滤
```
//将低于某个阈值的数据设置为一个固定值
NewSeismic = If (Seismic1 < Threshold, NewValue, Seismic1)

//将大于某个阈值的数据设置为一个固定值
NewSeismic = If (Seismic1 > Threshold, NewValue, Seismic1)

//多阈值筛选
NewSeismic = If (Seismic1 < A, Value1, If (Seismic1 > B, Value2, Seismic1))

```
