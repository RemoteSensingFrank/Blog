---
title: global rotation average
date: 2017-07-17 22:44:04
tags: 数学
categories: 数学
mathjax: true
---
为了简单就分到数学类中了，主要是说明了如何通过全局旋转矩阵的方法求解出整体的旋转矩阵，为什么要用这样的方法进行求解呢，最主要的原因在于采取此种方法进行求解不需要进行光束法平差就能够求得所有相机的旋转矩阵，这个应该是此类方法的最大的优点了，昨晚看得比较晚，但是基本概念还是懂了，今天结合OpenMVG进行一个记录，同时更进一步的加深理解，好了废话不多说，下面我们进入正题
做过CV或者摄影测量的人都应该知道，我们做摄影测量也好，做CV或者SFM也好，前面的步骤都大概是，影像获取，影像匹配，求影像间的相对变换关系，相信上面所说的几个步骤只要是做过影像匹配的人很简单就能够明白，现在我们面临最大的问题在于两两之间的相对关系求解出来之后怎么办，实际上面对这样的情况有两种方案，第一种就是类似摄影测量中相对定向的方法，连续进行模型之间的相对定向，将模型坐标统一起来，此外就是我们今天介绍的方法了，下一次再详细探讨在机器视觉和摄影测量中通过连续法进行定向的区别与联系
好了相信看这篇文章的大多是具有一定的基础，所以有关基础的东西就不讲了，直接从旋转矩阵开始,我们假设全局的旋转矩$R=[R_1,R_2,...R_i,R_j]$ 而任意两个旋转矩阵之间的关系我们可以看作是求解的相对元素，则有:$R=R_iR_j^{-1}\forall \in \varepsilon$其中 $\varepsilon$为影像集，从上式我们可以建立一个全局的旋转矩阵和相对的旋转矩阵之间的管理，当然对于一个闭环来说，这样的一个集合我们又称之为特殊正交组$SO(3)$,为什么要提出这个概念为什么要到这提出正交组这个概念，因为正交组有很多特性，根据这些特性可以进行进一步的推导。有了上面的相对变换的旋转矩阵和绝对变换的旋转矩阵之后根据下式就可以进行求解了  
$arg min_u \sum_1^n(1)$  
(1)式为求解的最小二乘的形式，求解上式的极小值则可以求解出全局的旋转矩阵，当然通过初始值，然后进行迭代求解。然而这样的求解过程依然稍显得有些复杂，我们进一步简化，不考虑旋转矩阵直接从旋转角出发进行考虑，假设全局的旋转角为$w=\{w_1,w_2,...w_i..w_j...\}$ 则相对的旋转角与全局旋转角的关系可以表示为：  
$w=w_i-w_j=[...-1,...,1...]\cdot w_g(2)$  
则根据式(2),可以列出全局的旋转角和相对旋转角之间的关系为：  
$Aw_g=w_r(3)$   
式(3)中A为值为0,-1,1的系数矩阵，通过求解上式可以得到旋转角进而得到旋转矩阵，总的来说就是这样，至于整个求解步骤哦我引用论文<sub>[1]</sub>中的求解步骤：
![enter image description here](https://lh3.googleusercontent.com/-lM2yyedHe8s/WW4UHH6TstI/AAAAAAAACPU/zXs1Sz4O2Zw3datEibwkPBdespAv2lLXQCLcBGAs/s0/global_rotation.png "global_rotation.png")  
当然论文中对求解方法进行了一定的改进，不过不管怎么改，总的来说还是万变不离其宗，至于各种的改进算法有兴趣可以自己进行更加深入的了解和学习，下面我们分析一下OpenMVG的做法。  
openMVG中计算GlobalRotation的代码为
```C
/// Compute from relative rotations the global rotations of the camera poses
bool GlobalSfMReconstructionEngine_RelativeMotions::Compute_Global_Rotations
(
  const rotation_averaging::RelativeRotations & relatives_R,
  Hash_Map<IndexT, Mat3> & global_rotations
)
{
  if(relatives_R.empty())
    return false;
  // Log statistics about the relative rotation graph
  {
    std::set<IndexT> set_pose_ids;
    for (const auto & relative_R : relatives_R)
    {
      set_pose_ids.insert(relative_R.i);
      set_pose_ids.insert(relative_R.j);
    }

    std::cout << "\n-------------------------------" << "\n"
      << " Global rotations computation: " << "\n"
      << "  #relative rotations: " << relatives_R.size() << "\n"
      << "  #global rotations: " << set_pose_ids.size() << std::endl;
  }

  // Global Rotation solver:
  const ERelativeRotationInferenceMethod eRelativeRotationInferenceMethod =
    TRIPLET_ROTATION_INFERENCE_COMPOSITION_ERROR;
    //TRIPLET_ROTATION_INFERENCE_NONE;

  system::Timer t;
  GlobalSfM_Rotation_AveragingSolver rotation_averaging_solver;
  const bool b_rotation_averaging = rotation_averaging_solver.Run(
    eRotation_averaging_method_, eRelativeRotationInferenceMethod,
    relatives_R, global_rotations);

  std::cout
    << "Found #global_rotations: " << global_rotations.size() << "\n"
    << "Timing: " << t.elapsed() << " seconds" << std::endl;


  if (b_rotation_averaging)
  {
    // Compute & display rotation fitting residual errors
    std::vector<float> vec_rotation_fitting_error;
    vec_rotation_fitting_error.reserve(relatives_R.size());
    for (const auto & relative_R : relatives_R)
    {
      const Mat3 & Rij = relative_R.Rij;
      const IndexT i = relative_R.i;
      const IndexT j = relative_R.j;
      if (global_rotations.count(i)==0 || global_rotations.count(j)==0)
        continue;
      const Mat3 & Ri = global_rotations[i];
      const Mat3 & Rj = global_rotations[j];
      const Mat3 eRij(Rj.transpose()*Rij*Ri);
      const double angularErrorDegree = R2D(getRotationMagnitude(eRij));
      vec_rotation_fitting_error.push_back(angularErrorDegree);
    }

    if (!vec_rotation_fitting_error.empty())
    {
      const float error_max = *max_element(vec_rotation_fitting_error.begin(), vec_rotation_fitting_error.end());
      Histogram<float> histo(0.0f,error_max, 20);
      histo.Add(vec_rotation_fitting_error.begin(), vec_rotation_fitting_error.end());
      std::cout
        << "\nRelative/Global degree rotations residual errors {0," << error_max<< "}:"
        << histo.ToString() << std::endl;
      {
        Histogram<float> histo(0.0f, 5.0f, 20);
        histo.Add(vec_rotation_fitting_error.begin(), vec_rotation_fitting_error.end());
        std::cout
          << "\nRelative/Global degree rotations residual errors {0,5}:"
          << histo.ToString() << std::endl;
      }
      std::cout << "\nStatistics about global rotation evaluation:" << std::endl;
      minMaxMeanMedian<float>(vec_rotation_fitting_error.begin(), vec_rotation_fitting_error.end());
    }

    // Log input graph to the HTML report
    if (!sLogging_file_.empty() && !sOut_directory_.empty())
    {
      // Log a relative pose graph
      {
        std::set<IndexT> set_pose_ids;
        Pair_Set relative_pose_pairs;
        for (const auto & view : sfm_data_.GetViews())
        {
          const IndexT pose_id = view.second->id_pose;
          set_pose_ids.insert(pose_id);
        }
        const std::string sGraph_name = "global_relative_rotation_pose_graph_final";
        graph::indexedGraph putativeGraph(set_pose_ids, rotation_averaging_solver.GetUsedPairs());
        graph::exportToGraphvizData(
          stlplus::create_filespec(sOut_directory_, sGraph_name),
          putativeGraph);

        using namespace htmlDocument;
        std::ostringstream os;

        os << "<br>" << sGraph_name << "<br>"
           << "<img src=\""
           << stlplus::create_filespec(sOut_directory_, sGraph_name, "svg")
           << "\" height=\"600\">\n";

        html_doc_stream_->pushInfo(os.str());
      }
    }
  }
  return b_rotation_averaging;
}
```
这一段代码不长，主要的处理函数为：rotation_averaging_solver
我们看看这个函数：
```C
bool GlobalSfM_Rotation_AveragingSolver::Run(
  ERotationAveragingMethod eRotationAveragingMethod,
  ERelativeRotationInferenceMethod eRelativeRotationInferenceMethod,
  const RelativeRotations & relativeRot_In,
  Hash_Map<IndexT, Mat3> & map_globalR
) const
{
  RelativeRotations relativeRotations = relativeRot_In;
  // We work on a copy, since inference can remove some relative motions

  switch(eRelativeRotationInferenceMethod)
  {
    case(TRIPLET_ROTATION_INFERENCE_NONE):
    break;
    case(TRIPLET_ROTATION_INFERENCE_COMPOSITION_ERROR):
    {
      //-------------------
      // Triplet inference (test over the composition error)
      //-------------------
      Pair_Set pairs = getPairs(relativeRotations);
      std::vector< graph::Triplet > vec_triplets = graph::tripletListing(pairs);
      //-- Rejection triplet that are 'not' identity rotation (error to identity > 5°)
      TripletRotationRejection(5.0f, vec_triplets, relativeRotations);

      pairs = getPairs(relativeRotations);
      const std::set<IndexT> set_remainingIds = graph::CleanGraph_KeepLargestBiEdge_Nodes<Pair_Set, IndexT>(pairs);
      if(set_remainingIds.empty())
        return false;
      KeepOnlyReferencedElement(set_remainingIds, relativeRotations);
    }
  break;
    default:
    std::cerr
      << "Unknown relative rotation inference method: "
      << (int) eRelativeRotationInferenceMethod << std::endl;
  }

  // Compute contiguous index (mapping between sparse index and contiguous index)
  //  from ranging in [min(Id), max(Id)] to  [0, nbCam]

  const Pair_Set pairs = getPairs(relativeRotations);
  Hash_Map<IndexT, IndexT> reindexForward, reindexBackward;
  reindex(pairs, reindexForward, reindexBackward);

  for(RelativeRotations::iterator iter = relativeRotations.begin();  iter != relativeRotations.end(); ++iter)
  {
    RelativeRotation & rel = *iter;
    rel.i = reindexForward[rel.i];
    rel.j = reindexForward[rel.j];
  }

  //- B. solve global rotation computation
  bool bSuccess = false;
  std::vector<Mat3> vec_globalR(reindexForward.size());
  switch(eRotationAveragingMethod)
  {
    case ROTATION_AVERAGING_L2:
    {
      //- Solve the global rotation estimation problem:
      bSuccess = rotation_averaging::l2::L2RotationAveraging(
        reindexForward.size(),
        relativeRotations,
        vec_globalR);
      //- Non linear refinement of the global rotations
      if (bSuccess)
        bSuccess = rotation_averaging::l2::L2RotationAveraging_Refine(
          relativeRotations,
          vec_globalR);

      // save kept pairs (restore original pose indices using the backward reindexing)
      for(RelativeRotations::iterator iter = relativeRotations.begin();  iter != relativeRotations.end(); ++iter)
      {
        RelativeRotation & rel = *iter;
        rel.i = reindexBackward[rel.i];
        rel.j = reindexBackward[rel.j];
      }
      used_pairs = getPairs(relativeRotations);
    }
    break;
    case ROTATION_AVERAGING_L1:
    {
      using namespace openMVG::rotation_averaging::l1;

      //- Solve the global rotation estimation problem:
      const size_t nMainViewID = 0; //arbitrary choice
      std::vector<bool> vec_inliers;
      bSuccess = rotation_averaging::l1::GlobalRotationsRobust(
        relativeRotations, vec_globalR, nMainViewID, 0.0f, &vec_inliers);

      std::cout << "\ninliers: " << std::endl;
      std::copy(vec_inliers.begin(), vec_inliers.end(), std::ostream_iterator<bool>(std::cout, " "));
      std::cout << std::endl;

      // save kept pairs (restore original pose indices using the backward reindexing)
      for (size_t i = 0; i < vec_inliers.size(); ++i)
      {
        if (vec_inliers[i])
        {
          used_pairs.insert(
            Pair(reindexBackward[relativeRotations[i].i],
                 reindexBackward[relativeRotations[i].j]));
        }
      }
    }
    break;
    default:
    std::cerr
      << "Unknown rotation averaging method: "
      << (int) eRotationAveragingMethod << std::endl;
  }

  if (bSuccess)
  {
    //-- Setup the averaged rotations
    for (size_t i = 0; i < vec_globalR.size(); ++i)  {
      map_globalR[reindexBackward[i]] = vec_globalR[i];
    }
  }
  else {
    std::cerr << "Global rotation solving failed." << std::endl;
  }

  return bSuccess;
}
```
从上面的函数可以看到，此函数的主要功能为根据average的方法进行计算并进行结果的精化处理，那么具体的计算应该就在GlobalRotationsRobuste中，我们继续查看这两个函数：
```c
// Robustly estimate global rotations from relative rotations as in:
// "Efficient and Robust Large-Scale Rotation Averaging", Chatterjee and Govindu, 2013
// and detect outliers relative rotations and return them with 0 in arrInliers
bool GlobalRotationsRobust(
  const RelativeRotations& RelRs,
  Matrix3x3Arr& Rs,
  const size_t nMainViewID,
  float threshold,
  std::vector<bool> * vec_Inliers)
{
  assert(!Rs.empty());

  // -- Compute coarse global rotation estimates:
  InitRotationsMST(RelRs, Rs, nMainViewID);

  // refine global rotations based on the relative rotations
  const bool bOk = RefineRotationsAvgL1IRLS(RelRs, Rs, nMainViewID);

  // find outlier relative rotations
  if (threshold>=0 && vec_Inliers)  {
    FilterRelativeRotations(RelRs, Rs, threshold, vec_Inliers);
  }

  return bOk;
} // GlobalRotationsRobust
//----------------------------------------------------------------
```
可以看到函数很简单，首先初始化得到初始的全局旋转矩阵，然后通过IRLS方法，也就是我们上文理论分析中提到的方法进行迭代精华求解，最后通过滤波的方法得到正确的全局旋转矩阵，如果大家能够找到这个函数，则所有包含的函数都在这个文件中，初始化的方法是通过最大生成树来实现的，生成最大生成树，然后以根节点为标准计算所有旋转角与标准的差异作为全局旋转角，最后进行迭代精华和滤去粗差的过程<sub>[2]</sub>  
[1]:Chatterjee, Avishek, and Venu Madhav Govindu. "Efficient and robust large-scale rotation averaging." Proceedings of the IEEE International Conference on Computer Vision. 2013.  
[2]:https://github.com/openMVG/openMVG/
